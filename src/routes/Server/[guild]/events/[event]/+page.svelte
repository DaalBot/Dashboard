<script lang="ts">
    import type monaco from "monaco-editor";
    import { onMount } from "svelte";
    import editorWorker from "monaco-editor/esm/vs/editor/editor.worker?worker";
    import jsonWorker from "monaco-editor/esm/vs/language/json/json.worker?worker";
    import cssWorker from "monaco-editor/esm/vs/language/css/css.worker?worker";
    import htmlWorker from "monaco-editor/esm/vs/language/html/html.worker?worker";
    import tsWorker from "monaco-editor/esm/vs/language/typescript/ts.worker?worker";
    import { modules, DBAPI } from "@daalbot/api";
    import { page } from "$app/state";
    import type { Event } from "@daalbot/api/dist/src/modules/events";

    let divEl: HTMLDivElement;
    let editor: monaco.editor.IStandaloneCodeEditor;
    let Monaco;
    let api: DBAPI;
    let saveButton: HTMLButtonElement;
    let lastSavedCode: string;
    let event: Event;

    onMount(async () => {
        api = new DBAPI({
            auth: {
                token: localStorage.getItem("accessToken") || "",
                type: "User",
            },
            guildId: page.params.guild!,
        });

        modules.getEvents(api).then((events) => {
            // @ts-ignore
            event = events.find((e) => e.id === page.params.event);
            if (!event) {
                alert("Event not found - modules.getEvents failed");
                window.location.href = "/Error?type=failed";
            }
        });

        // @ts-ignore
        self.MonacoEnvironment = {
            getWorker: function (_moduleId: any, label: string) {
                if (label === "json") {
                    return new jsonWorker();
                }
                if (label === "css" || label === "scss" || label === "less") {
                    return new cssWorker();
                }
                if (
                    label === "html" ||
                    label === "handlebars" ||
                    label === "razor"
                ) {
                    return new htmlWorker();
                }
                if (label === "typescript" || label === "javascript") {
                    return new tsWorker();
                }
                return new editorWorker();
            },
        };

        lastSavedCode = await modules.readEventCode(api, page.params.event!);

        // Wait for fonts to load
        await document.fonts.ready;

        Monaco = await import("monaco-editor");
        editor = Monaco.editor.create(divEl, {
            value: lastSavedCode,
            language: "javascript",
            theme: "vs-dark",
            fontFamily:
                '"JetBrains Mono", "Consolas", "Courier New", monospace',
            fontWeight: "400",
            fontLigatures: true,
            fontSize: 14,
        });
    });

    onMount(() => {
        return () => {
            if (editor) {
                editor.dispose();
            }
        };
    });

    async function saveCode() {
        if (!editor) return;
        const code = editor.getValue();

        if (code === lastSavedCode) {
            saveButton.innerText = "No changes made";
            setTimeout(() => {
                saveButton.innerText = "Save";
            }, 2000);
            return;
        }

        try {
            await modules.writeEventCode(api, page.params.event!, code);
            saveButton.innerText = "Saved!"; // We don't want to freeze the page with alert();
            setTimeout(() => {
                saveButton.innerText = "Save";
            }, 2000);
        } catch (err) {
            console.error(err);
            alert("Failed to save code");
        }
    }
</script>

<svelte:head>
    <title>
        Editor - {event?.name}
    </title>
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link
        rel="preconnect"
        href="https://fonts.gstatic.com"
        crossorigin="anonymous"
    />
    <link
        href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@100..800&display=swap"
        rel="stylesheet"
    />
</svelte:head>

<!-- svelte-ignore a11y_no_static_element_interactions -->
<div
    bind:this={divEl}
    class="h-screen jetbrains-mono-code"
    on:keydown={(e) => {
        if (e.ctrlKey && e.key === "s") {
            e.preventDefault();

            saveCode();
        }
    }}
></div>

<button bind:this={saveButton} on:click={saveCode}> Save </button>

<style>
    .h-screen {
        height: 100vh;
    }

    .jetbrains-mono-code {
        font-family: "JetBrains Mono", monospace;
        font-optical-sizing: auto;
        font-weight: 400;
        font-style: normal;
    }

    button {
        position: relative;
        width: 100%;
        bottom: 0;
        right: 1rem;
        background-color: #4caf50;
        border: none;
        color: white;
        padding: 15px 32px;
        text-align: center;
        text-decoration: none;
        display: inline-block;
        font-size: 16px;
        margin-top: 1rem;
        margin: 4px 2px;
        cursor: pointer;
    }
</style>
