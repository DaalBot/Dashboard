<script lang="ts">
    import type { Event } from "@daalbot/api/dist/src/modules/events";
    
    import { onMount } from "svelte";
    import { page } from "$app/state";

    import { DBAPI, modules } from "@daalbot/api";
    import type { GatewayEvent } from "@daalbot/api/dist/src/types";
    import { browser } from "$app/environment";

    let api: DBAPI;
    let events: Event[] = [];
    let description = '';
    let name = '';
    let trigger: GatewayEvent;
    let validTriggers: Array<{ name: string, value: GatewayEvent }> = [
        { name: 'Message Created', value: 'messageCreate' }, // Load default value while we fetch the real data
    ];

    onMount(async() => {
        api = new DBAPI({
            auth: {
                token: localStorage.getItem('accessToken') || '',
                type: 'User'
            },
            guildId: page.params.guild
        });

        validTriggers = await fetch('https://cdn.jsdelivr.net/gh/DaalBot/API/config/events.json')
            .then(res => res.json())
            .then(data => data.event_types);

        events = await modules.getEvents(api);
    })
</script>

<form on:submit={async() => {
    if (!browser) return;
    if (!name || !description || !trigger) return alert('Please fill in all fields');
    if (events.find(e => e.name === name) !== undefined) return alert('Event with this name already exists');
    if (!validTriggers.map(t=>t.value).includes(trigger)) return alert('Invalid trigger');

    const newEvent = await modules.createEvent(
        api,
        trigger,
        name,
        description
    );
    
    alert(`Event created with ID: ${newEvent.id}`);

    window.location.href = `/Server/${page.params.guild}/events`;
}}>
    <input type="text" class="input" bind:value={name} placeholder="Enter event name" />
    <input type="text" class="input" bind:value={description} placeholder="Enter event description" />
    <select bind:value={trigger} required>
        <option value="" disabled selected>Select a trigger</option>
        {#each validTriggers as { name, value }}
            <option value={value}>{name}</option>
        {/each}
    </select>
    <button type="submit">Submit</button>
</form>

<style>
    input:first-child {
        margin-top: 0;
    }

    input {
        background-color: #2f2f2f;
        color: white;
        border: none;
        padding: 0.75rem 1rem;
        border-radius: 0.5rem;
        width: 98.4%;
        margin-top: 1rem;
    }

    button {
        background-color: #826ae3;
        color: white;
        border: none;
        padding: 0.5rem 1rem;
        border-radius: 0.5rem;
        cursor: pointer;
        font-weight: bolder;
        font-size: 1.25rem;
        transition: background-color 0.3s ease;
        margin-top: 1rem;
        width: 100%;

        flex: 1;
    }

    select {
        background-color: #2f2f2f;
        color: white;
        border: none;
        padding: 0.75rem 1rem;
        border-radius: 0.5rem;
        width: 100%;
        margin-top: 1rem;
    }
    select:focus {
        outline: none;
        box-shadow: 0 0 0 2px #826ae3;
    }
    select::placeholder {
        color: #7a7a7a;
    }
</style>
