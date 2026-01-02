<!-- TODO: THIS PAGE ISN'T DONE YET, GOT SOME SERVER SIDE ISSUES??????? -->
<script lang="ts">
    import { browser } from "$app/environment";
    import { page } from "$app/state";
    import { DBAPI, modules } from "@daalbot/api";
    import { onMount } from "svelte";

    let api: DBAPI;
    let showLogModal = '';
    let dialogElement: HTMLDialogElement;

    onMount(async() => {
        api = new DBAPI({
            auth: {
                token: localStorage.getItem("accessToken") || "",
                type: "User",
            },
            guildId: page.params.guild!,
        });

        onHashLoad();

        window.addEventListener('hashchange', () => {
            onHashLoad();
        });

        dialogElement.onclose = () => {
            window.location.hash = '';
            showLogModal = '';
        };
    });

    function onHashLoad() {
        const hash = page.url.hash.substring(1);
        if (!hash) {
            showLogModal = '';
            if (dialogElement) {
                dialogElement.close();
            }
            return;
        }

        if (hash.startsWith('logs-')) {
            const level = hash.split('-')[1];
            
            showLogModal = level;

            if (showLogModal && dialogElement) {
                dialogElement.showModal();
            } else if (!showLogModal && dialogElement) {
                dialogElement.close();
            }
        }
    }
</script>

{#snippet locationButton(href: string, text: string)}
    <a class="button" href={href}>{text}</a>
{/snippet}

<main inert={!!showLogModal}>
    <h1>Debug Tools</h1>
    <h2>Logging</h2>
    {@render locationButton('#logs-info', 'Info')}
    {@render locationButton('#logs-error', 'Error')}
    {@render locationButton('#logs-debug', 'Debug')}
</main>

<dialog bind:this={dialogElement} closedby="any">
    {#if showLogModal}
        <h2>{showLogModal.charAt(0).toUpperCase() + showLogModal.slice(1)} Logs</h2>
        <pre>
            {#await modules.readDatabase(api, `logs/${showLogModal}`)}
                Loading...
            {:then data}
                {data}
            {:catch error}
                Error loading logs: {error.message}
            {/await}
        </pre>
    {/if}
</dialog>

<style>
    main {
        padding: 1rem;
    }

    h1 {
        font-family: 'Lexend', sans-serif;
        font-weight: 600;
        font-size: 2rem;
        margin-bottom: 0.5rem;
    }

    h2 {
        font-family: 'Lexend', sans-serif;
        font-weight: 600;
        font-size: 1.5rem;
        margin-top: 1rem;
        margin-bottom: 0.75rem;
    }

    .button {
        background-color: #2f2f2f;
        color: white;
        padding: 0.5rem 1rem;
        margin-right: 0.5rem;
        text-decoration: none;
        border-radius: 4px;
        border: none;
        cursor: pointer;
    }

    .button:hover {
        background-color: #646464;
    }

    dialog {
        background-color: #1f1f1f;
        border-radius: 8px;
        padding: 1rem;
    }

    dialog::backdrop {
        background-color: rgba(0, 0, 0, 0.5);
    }
</style>