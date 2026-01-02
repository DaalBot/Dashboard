<script lang="ts">
    import { onMount } from 'svelte';
    import { page } from '$app/state';
    import { browser } from '$app/environment';
    import { DBAPI, modules } from '@daalbot/api';
    import type { Event } from '@daalbot/api/dist/src/modules/events';
    import { truncateStr } from '$lib/tools';
    import { deleteEventVariable } from '@daalbot/api/dist/src/modules';

    let events: Array<Event> = [];
    let shouldRenderEventModal = '';
    let shouldRenderVariableModal = '';
    let api: DBAPI;
    let mode: 'events' | 'variables' = page.url.hash.includes('variables') ? 'variables' : 'events';
    let variableScope = page.url.searchParams.get('varscope') || 'global';
    let variables: { name: string, value: string }[] = [];
    
    onMount(async() => {
        api = new DBAPI({
            auth: { 
                token: localStorage.getItem('accessToken') || '', 
                type: 'User' 
            },
            guildId: page.params.guild!
        });

        events = await modules.getEvents(api);
        variables = await modules.getEventVariable(api, variableScope) as {name: string, value: string}[];
    })
</script>

<svelte:head>
    <title>
        Events
    </title>
</svelte:head>

{#snippet eventModal(event: Event)}
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div class="modal">
        <div class="modal-content">
            <button on:click={() => { shouldRenderEventModal = ''; }}>Close</button>
            <h2>{event.name}</h2>
            <p>{event.description}</p>
            <p>Enabled: {event.enabled ? 'Yes' : 'No'}</p>
            <p>On: {event.on}</p>
            <p>ID: {event.id}</p>

            <button on:click={() => {
                modules.setEventState(api, event.id, !event.enabled).then(() => {
                    shouldRenderEventModal = '';
                    events = events.map(e => e.id === event.id ? { ...e, enabled: !e.enabled } : e);
                }).catch(err => {
                    console.error(err);
                    alert('Failed to update event state');
                });
            }}>
                {event.enabled ? 'Disable' : 'Enable'}
            </button>
            <button on:click={() => {
                if (!browser) return;
                window.location.href = `/Server/${page.params.guild}/events/${event.id}`;
            }}>
                Edit
            </button>
            <button on:click={() => {
                modules.delEvent(api, event.id).then(() => {
                    shouldRenderEventModal = '';
                    events = events.filter(e => e.id !== event.id);
                }).catch(err => {
                    console.error(err);
                    alert('Failed to delete event');
                });
            }}>
                Delete
            </button>
        </div>
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <div class="modal-overlay" on:click={() => { shouldRenderEventModal = ''; }}></div>
        <style>
            .modal {
                position: fixed;
                top: 0;
                left: 0;
                width: 100%;
                height: 100%;
                display: flex;
                justify-content: center;
                align-items: center;
                z-index: 1000;
            }
            .modal-content {
                background-color: #2f2f2f;
                padding: 1rem;
                border-radius: 0.5rem;
                z-index: 1001;
            }
            .modal-overlay {
                position: absolute;
                top: 0;
                left: 0;
                z-index: 999;
                width: 100%;
                height: 100%;
                background-color: rgba(0, 0, 0, 0.5);
            }

            button:first-child {
                margin-top: 0;
            }

            button {
                background-color: #3f3f3f;
                border: none;
                border-radius: .75rem;
                width: 100%;
                padding: 0.5rem 2rem;

                cursor: pointer;

                margin-top: 1rem;
            }
        </style>
    </div>
{/snippet}

{#snippet variableModal(variable: { name: string, value: string })}
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div class="modal">
        <div class="modal-content">
            <button on:click={() => { shouldRenderVariableModal = ''; }}>Close</button>
            <h2>Name: {variable.name}</h2>
            <p>
                <strong>Value:</strong> {variable.value}
            </p>
            <button on:click={() => {
                if (!browser) return;
                const newValue = prompt('Enter new value for variable:', variable.value);
                if (newValue === null) return; // User cancelled
                modules.setEventVariable(api, variableScope, variable.name, newValue).then(() => {
                    shouldRenderVariableModal = '';
                    variables = variables.map(v => v.name === variable.name ? { ...v, value: newValue } : v);
                }).catch(err => {
                    console.error(err);
                    alert('Failed to update variable');
                });
            }}>
                Edit
            </button>
            <button on:click={() => {
                if (!browser) return;
                deleteEventVariable(api, variableScope, variable.name).then(() => {
                    shouldRenderVariableModal = '';
                    variables = variables.filter(v => v.name !== variable.name);
                }).catch(err => {
                    console.error(err);
                    alert('Failed to delete variable');
                });
            }}>
                Delete
            </button>
        </div>
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <div class="modal-overlay" on:click={() => { shouldRenderVariableModal = ''; }}></div>
        <style>
            .modal {
                position: fixed;
                top: 0;
                left: 0;
                width: 100%;
                height: 100%;
                display: flex;
                justify-content: center;
                align-items: center;
                z-index: 1000;
            }
            .modal-content {
                background-color: #2f2f2f;
                padding: 1rem;
                border-radius: 0.5rem;
                z-index: 1001;
            }
            .modal-overlay {
                position: absolute;
                top: 0;
                left: 0;
                z-index: 999;
                width: 100%;
                height: 100%;
                background-color: rgba(0, 0, 0, 0.5);
            }

            button:first-child {
                margin-top: 0;
            }

            button {
                background-color: #3f3f3f;
                border: none;
                border-radius: .75rem;
                width: 100%;
                padding: 0.5rem 2rem;

                cursor: pointer;

                margin-top: 1rem;
            }
        </style>
    </div>
{/snippet}

<main>
    <div class="switch">
        <button on:click={() => { mode = 'events'; }} class:active={mode === 'events'}>
            Events
        </button>
        <button on:click={() => { mode = 'variables'; }} class:active={mode === 'variables'}>
            Variables
        </button>
    </div>
    <button on:click={() => {
        if (!browser) return;
        window.location.href = `/Server/${page.params.guild}/events/new/${mode.slice(0, -1)}`;
    }}>
        Create
    </button>
    {#if mode == 'events'}
    {#each events as event}
        <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
        <div class="event" style="background-color: {event.enabled ? '#2f2f2f' : '#ff3400'};" on:click={() => { shouldRenderEventModal = event.id; }}>
            <h3>{event.name} ({event.on})</h3>
            <p>{truncateStr(event.description, 128)}</p>
        </div>
        {#if shouldRenderEventModal === event.id}
            {@render eventModal(event)}
        {/if}
    {/each}
    {:else if mode == 'variables'}
    <select bind:value={variableScope} on:change={() => {
        if (!browser) return;
        variables = [];
        modules.getEventVariable(api, variableScope).then(vars => {
            variables = vars as {name: string, value: string}[];
        }).catch(err => {
            console.error(err);
            alert('Failed to fetch variables');
        });
    }}>
        <option value="global">Global</option>
        {#each events as event}
            <option value={event.id}>{event.name} - {event.description} [{event.id}]</option>
        {/each}
    </select>
    <div class="variables">
        {#each variables as variable}
            <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
            <div class="variable" on:click={() => {
                if (!browser) return;
                shouldRenderVariableModal = variable.name;
            }}>
                <h3>{variable.name}</h3>
                <p>{variable.value}</p>
            </div>
            {#if shouldRenderVariableModal === variable.name}
                {@render variableModal(variable)}
            {/if}
        {/each}
    </div>
    {:else}
    <h3>Invalid mode.. How did you get here?</h3>
    {/if}
</main>

<style>
    main {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .event {
        background-color: #2f2f2f;
        padding: 1rem;
        border-radius: 0.5rem;
    }

    .event h3 {
        margin: 0;
    }

    .switch {
        display: flex;
        flex-direction: row;
        gap: 1rem;
        justify-content: center;
        align-items: center;
        margin-bottom: 1rem;
    }

    button {
        background-color: #4a4a4a;
        color: white;
        border: none;
        padding: 0.75rem 1rem;
        border-radius: 0.5rem;
        cursor: pointer;
        font-weight: bolder;
        font-size: 1.25rem;
        transition: background-color 0.3s ease;

        flex: 1;
    }

    .switch button.active {
        background-color: #826ae3;
        color: white;
    }

    .variables {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .variable {
        background-color: #2f2f2f;
        padding: 1rem;
        border-radius: 0.5rem;
    }

    .variable h3 {
        margin: 0;
    }

    select {
        background-color: #2f2f2f;
        color: white;
        border: none;
        padding: 0.5rem 1rem;
        border-radius: 0.5rem;
        width: 98.4%;
        font-size: 1rem;
    }

    select:focus {
        outline: none;
        box-shadow: 0 0 0 2px #826ae3;
    }

    select::placeholder {
        color: #7a7a7a;
    }

    select:disabled {
        background-color: #3f3f3f;
        color: #7a7a7a;
    }
    
    select:disabled:focus {
        box-shadow: none;
    }
</style>