<script lang="ts">
    import { onMount, setContext } from "svelte";
    import LogPane from "./LogPane.svelte";
    import RevisionPane from "./RevisionPane.svelte";
    import BoundQuery from "./controls/BoundQuery.svelte";
    import IdSpan from "./controls/IdSpan.svelte";
    import { onEvent, type Query, query, trigger } from "./ipc.js";
    import type { InputRequest } from "./messages/InputRequest";
    import type { InputResponse } from "./messages/InputResponse";
    import type { RepoConfig } from "./messages/RepoConfig";
    import type { RevId } from "./messages/RevId";
    import type { RevResult } from "./messages/RevResult";
    import ChangeMutator from "./mutators/ChangeMutator";
    import RefMutator from "./mutators/RefMutator";
    import RevisionMutator from "./mutators/RevisionMutator";
    import Zone from "./objects/Zone.svelte";
    import ErrorDialog from "./shell/ErrorDialog.svelte";
    import InputDialog from "./shell/InputDialog.svelte";
    import ModalOverlay from "./shell/ModalOverlay.svelte";
    import Pane from "./shell/Pane.svelte";
    import RecentWorkspaces from "./shell/RecentWorkspaces.svelte";
    import type Settings from "./shell/Settings";
    import StatusBar from "./shell/StatusBar.svelte";
    import {
        currentContext,
        currentInput,
        currentMutation,
        repoConfigEvent,
        repoStatusEvent,
        revisionSelectEvent,
    } from "./stores.js";

    let selection: Query<RevResult> = $state({ type: "wait" });
    // for open recent workspaces when error dialogs happen
    let recentWorkspaces: string[] = $state([]);

    document.addEventListener("keydown", (event) => {
        if (event.key === "o" && event.ctrlKey) {
            event.preventDefault();
            trigger("forward_accelerator", { key: "o" });
        }
    });

    document.body.addEventListener("click", () => currentContext.set(null), true);

    // this is a special case - most triggers are fire-and-forget, but we really need a
    // vizjj://repo/config event in response to this one. if it takes too long, we make our own
    trigger("notify_window_ready");
    let loadTimeout: number | null;
    onMount(() => {
        if ($repoConfigEvent.type == "Initial") {
            loadTimeout = setTimeout(() => {
                repoConfigEvent.set({ type: "TimeoutError" });
            }, 10_000);
        }
    });

    let settings: Settings = {
        markUnpushedBranches: true,
    };
    setContext<Settings>("settings", settings);

    onEvent("vizjj://context/revision", mutateRevision);
    onEvent("vizjj://context/tree", mutateTree);
    onEvent("vizjj://context/branch", mutateRef);
    onEvent("vizjj://input", requestInput);

    $effect(() => {
        if ($repoConfigEvent) loadRepo($repoConfigEvent);
    });
    $effect(() => {
        if ($repoStatusEvent && $revisionSelectEvent) loadChange($revisionSelectEvent.id);
    });
    $effect(() => {
        if (["LoadError", "TimeoutError", "WorkerError"].includes($repoConfigEvent.type)) {
            queryRecentWorkspaces();
        }
    });

    async function loadRepo(config: RepoConfig) {
        if (loadTimeout) {
            clearTimeout(loadTimeout);
            loadTimeout = null;
        }

        $revisionSelectEvent = undefined;
        if (config.type == "Workspace") {
            settings.markUnpushedBranches = config.mark_unpushed_branches;
            $repoStatusEvent = config.status;
        }
    }

    async function loadChange(id: RevId) {
        let rev = await query<RevResult>("query_revision", { id }, (q) => (selection = q));
        if (
            rev.type == "data" &&
            rev.value.type == "NotFound" &&
            id.commit.hex != $repoStatusEvent?.working_copy.hex
        ) {
            return loadChange({
                change: { type: "ChangeId", hex: "@", prefix: "@", rest: "" },
                commit: $repoStatusEvent!.working_copy,
            });
        }

        selection = rev;
    }

    async function queryRecentWorkspaces() {
        const result = await query<string[]>("query_recent_workspaces", null);
        recentWorkspaces = result.type === "data" ? result.value : [];
    }

    function mutateRevision(event: string) {
        if ($currentContext?.type == "Revision") {
            new RevisionMutator($currentContext.header).handle(event);
        }
        $currentContext = null;
    }

    function mutateTree(event: string) {
        if ($currentContext?.type == "Change") {
            new ChangeMutator($currentContext.header, $currentContext.path).handle(event);
        }
        $currentContext = null;
    }

    function mutateRef(event: string) {
        if ($currentContext?.type == "Ref") {
            new RefMutator($currentContext.ref).handle(event);
        }
        $currentContext = null;
    }

    function requestInput(event: InputRequest) {
        $currentInput = Object.assign(event, {
            callback: (response: InputResponse) => {
                $currentInput = null;
                trigger("notify_input", { response });
            },
        });
    }
</script>

<Zone operand={{ type: "Repository" }} alwaysTarget>
    {#snippet children({ target })}
        <div
            id="shell"
            class={$repoConfigEvent?.type == "Workspace" ? $repoConfigEvent.theme_override : ""}>
            {#if $repoConfigEvent.type == "Initial"}
                <Pane>
                    {#snippet header()}
                        <h2>Loading...</h2>
                    {/snippet}
                </Pane>
                <div class="separator"></div>
                <Pane />
            {:else if $repoConfigEvent.type == "Workspace"}
                {#key $repoConfigEvent.absolute_path}
                    <LogPane
                        default_query={$repoConfigEvent.default_query}
                        latest_query={$repoConfigEvent.latest_query} />
                {/key}

                <div class="separator"></div>

                <BoundQuery query={selection}>
                    {#snippet children({ data })}
                        {#if data.type == "Detail"}
                            <RevisionPane rev={data} />
                        {:else}
                            <Pane>
                                {#snippet header()}
                                    <h2>Not Found</h2>
                                {/snippet}
                                {#snippet body()}
                                    <p>
                                        Revision <IdSpan id={data.id.change} />
                                        |<IdSpan id={data.id.commit} /> does not exist.
                                    </p>
                                {/snippet}
                            </Pane>
                        {/if}
                    {/snippet}
                    {#snippet error({ message })}
                        <Pane>
                            {#snippet header()}
                                <h2>Error</h2>
                            {/snippet}
                            {#snippet body()}
                                <p>{message}</p>
                            {/snippet}
                        </Pane>
                    {/snippet}
                    {#snippet wait()}
                        <Pane>
                            {#snippet header()}
                                <h2>Loading...</h2>
                            {/snippet}
                        </Pane>
                    {/snippet}
                </BoundQuery>
            {:else if $repoConfigEvent.type == "LoadError"}
                <ModalOverlay>
                    <ErrorDialog title="No Workspace Loaded">
                        <p>{$repoConfigEvent.message}.</p>
                        <p>Try opening a workspace from the Repository menu.</p>
                        <RecentWorkspaces workspaces={recentWorkspaces} />
                    </ErrorDialog>
                </ModalOverlay>
            {:else if $repoConfigEvent.type == "TimeoutError"}
                <ModalOverlay>
                    <ErrorDialog title="No Workspace Loaded" severe>
                        <p>Error communicating with backend: the operation is taking too long.</p>
                        <p>You may need to restart vizjj to continue.</p>
                        <RecentWorkspaces workspaces={recentWorkspaces} />
                    </ErrorDialog>
                </ModalOverlay>
            {:else}
                <ModalOverlay>
                    <ErrorDialog title="Fatal Error" severe>
                        <p>Error communicating with backend: {$repoConfigEvent.message}.</p>
                        <p>You may need to restart vizjj to continue.</p>
                        <RecentWorkspaces workspaces={recentWorkspaces} />
                    </ErrorDialog>
                </ModalOverlay>
            {/if}

            <div class="separator" style="grid-area: 2/1/3/4"></div>

            <StatusBar {target} />

            {#if $currentInput}
                <ModalOverlay>
                    <InputDialog
                        title={$currentInput.title}
                        detail={$currentInput.detail}
                        fields={$currentInput.fields}
                        response={(event) => $currentInput?.callback(event)} />
                </ModalOverlay>
            {:else if $currentMutation}
                <ModalOverlay>
                    {#if $currentMutation.type == "data" && ($currentMutation.value.type == "InternalError" || $currentMutation.value.type == "PreconditionError")}
                        <ErrorDialog
                            title="Command Error"
                            onClose={() => ($currentMutation = null)}
                            severe>
                            {#if $currentMutation.value.type == "InternalError"}
                                <p>
                                    {#each $currentMutation.value.message.lines as line}
                                        {line}<br />
                                    {/each}
                                </p>
                            {:else}
                                <p>{$currentMutation.value.message}</p>
                            {/if}
                        </ErrorDialog>
                    {:else if $currentMutation.type == "error"}
                        <ErrorDialog
                            title="IPC Error"
                            onClose={() => ($currentMutation = null)}
                            severe>
                            <p>{$currentMutation.message}</p>
                        </ErrorDialog>
                    {/if}
                </ModalOverlay>
            {/if}
        </div>
    {/snippet}
</Zone>

<style>
    #shell {
        width: 100vw;
        height: 100vh;

        display: grid;
        grid-template-columns: 1fr 3px 1fr;
        grid-template-rows: 1fr 3px 30px;
        grid-template-areas:
            "content content content"
            ". . ."
            "footer footer footer";

        background: var(--ctp-crust);
        color: var(--ctp-text);

        user-select: none;
    }

    .separator {
        background: var(--ctp-overlay0);
    }
</style>
