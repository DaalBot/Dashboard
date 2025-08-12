<script lang="ts">
    import { page } from "$app/state";
    import Slider from "$lib/components/Slider.svelte";
    import { DBAPI, modules } from "@daalbot/api";
    import type { CurrentGuildResponse } from "@daalbot/api/dist/src/modules/currentGuild";
    import type { InviteTrackingData } from "@daalbot/api/dist/src/modules/inviteTracking";
    import type { RoleInviteData } from "@daalbot/api/dist/src/modules/roleInvites";
    import { onMount } from "svelte";

    let api: DBAPI;
    let guild: CurrentGuildResponse;

    let inviteTrackingState: InviteTrackingData = { enabled: false, invites: [] };
    let roleInviteLinksState: RoleInviteData = { enabled: false, links: {} };
    let codeToAdd: string = '';
    let roleToAdd: string = '';
    let addRoleMenu: HTMLElement | null = null;
    let addRoleBusy: boolean = false;
    let deleteRoleBusy: boolean = false;

    async function deleteRole(code: string, roleId: string) {
        if (deleteRoleBusy) return; // Prevent multiple clicks
        deleteRoleBusy = true;
        let roleName = guild.roles.find(r => r.id === roleId)?.name;
        if (!roleName)
            roleName = 'Unknown Role (ID: ' + roleId + ')';

        if (confirm(`Are you sure you want to delete the role "${roleName}" from the invite code "${code}"?`)) {
            addRoleBusy = true;
            try {
                await modules.deleteRoleLink(api, code, roleId);
                window.location.reload();
            } catch (error) {
                console.error("Failed to delete role from invite:", error);
                alert("Failed to delete role from invite. Please try again later.");
            }

            addRoleBusy = false;
            deleteRoleBusy = false;
        }
    }

    onMount(async () => {
        const guildId = page.params.guild ?? "THIS_IS_BUGGED";
        api = new DBAPI({
            auth: {
                token: localStorage.getItem("accessToken") || "",
                type: "User",
            },
            guildId
        });

        guild = await modules.getCurrentGuild(api);

        try {
            inviteTrackingState = await modules.getInviteTrackingData(api);
            roleInviteLinksState = await modules.getRoleLinks(api);
        } catch (error) {
            console.error("Failed to fetch invite tracking state:", error);
        }
    });
</script>

<h1>Role Invites</h1>

<h2>Should role invites be enabled?</h2>
{#if roleInviteLinksState.enabled && !inviteTrackingState.enabled}
    <p>You have role invites enabled, but invite tracking is disabled; please enable invite tracking to use role invites.</p>
{/if}
<Slider
    checked={roleInviteLinksState.enabled}
    onChange={async (event) => {
        console.log("Role invites toggle changed:", event.currentTarget.checked);
        await modules.toggleRoleLinks(api, event.currentTarget.checked);

        if (!inviteTrackingState.enabled) {
            const fixInvalidState = confirm(
                `The invite tracking module is currently disabled, which is required for role invites to function properly. Would you like to enable invite tracking now?`
            );
            if (fixInvalidState) {
                await modules.toggleInviteTracking(api, true);
            }
        }
    }}
/>

<h2>Role Invites</h2>
<table>
    <thead>
        <tr>
            <th>Code</th>
            <th>Roles</th>
        </tr>
    </thead>
    <tbody>
        {#if Object.keys(roleInviteLinksState.links).length > 0}
        {#each Object.entries(roleInviteLinksState.links) as [code, roles]}
            <tr>
                <td>{code}</td>
                <td class="roles">
                    {#if roles.length > 0}
                        {#each roles as role}
                            <sl-visually-hidden>
                                <button on:click={async() => {await deleteRole(code, role)}}>
                                    <span>Delete role {guild.roles.find(r => r.id === role)?.name} from invite {code}</span>
                                </button>
                            </sl-visually-hidden>
                            <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
                            <span class="role" on:click={async() => {await deleteRole(code, role)}}>{guild.roles.find(r => r.id === role)?.name}</span>
                        {/each}
                    {:else}
                        <span class="no-roles">No roles assigned</span>
                    {/if}
                </td>
            </tr>
        {/each}
        {:else}
            <tr>
                <td colspan="2">No role invites found.</td>
            </tr>
        {/if}
        <tr>
            <td colspan="2">
                <button on:click={() => {
                    if (addRoleMenu) addRoleMenu.style.display = 'block'
                    else alert('Failed to open add role invite menu, please try again later.');
                }}>Add Role Invite</button>
            </td>
        </tr>
    </tbody>
</table>

<div class="add-role-menu" bind:this={addRoleMenu}>
    {#if !addRoleBusy}
    <h1>Add Role Invite</h1>
    <p>Adding a existing code will simply add a new role to the invite.</p>
    <input type="text" placeholder="Enter a invite code" bind:value={codeToAdd} />
    <select bind:value={roleToAdd}>
        <option value="" disabled selected>Select a role</option>
        {#each guild?.roles?.sort((a, b) => a.name.localeCompare(b.name)) as role}
            <option value={role.id}>{role.name}</option>
        {/each}
    </select>
    <button on:click={async () => {
        if (codeToAdd && roleToAdd) {
            await modules.createRoleLink(api, codeToAdd, roleToAdd);

            window.location.reload();
        } else {
            alert('Please enter a invite code and select a role to add.');
        }
    }}>Add</button>
    {:else}
        <h1>Applying...</h1>
    {/if}
</div>

<style>
    table {
        width: 100%;
        border-collapse: collapse;
        margin-top: 1em;

        background-color: #2f2f2f;

        border-radius: 1em;
        overflow: hidden;
    }

    th, td {
        padding: 1.5em;
        text-align: left;
    }

    tr {
        cursor: pointer;

        transition: background-color 0.3s;
    }

    tr:hover {
        background-color: #3f3f3f;
    }

    th {
        background-color: #3f3f3f;
    }

    button {
        background-color: #826ae3;
        color: white;
        border: none;
        font-size: 1.2em;
        padding: 0.5em 1em;
        border-radius: 0.5em;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }

    .add-role-menu {
        margin-top: 2em;
        padding: 1em;
        background-color: #3f3f3f;
        border-radius: 1em;

        display: none;
    }

    select {
        background-color: #2f2f2f;
        color: white;
        margin-top: 0;
        margin-bottom: 1em;

        border: 1px solid #444;
        padding: .5em;
        font-size: 1.5em;
        border-radius: 0.5em;

        width: 95%;
    }

    input[type="text"] {
        width: 10em;
        padding: 0.5em;
        margin: 1em 0.25em;
        margin-bottom: .25em;
        border: 1px solid #444;
        background-color: #2f2f2f;
        color: white;
        font-size: 1.5em;
        border-radius: 0.5em;
    }

    .roles {
        display: flex;
        flex-wrap: wrap;
        gap: 0.5em;
    }

    .roles .role {
        background-color: #4f4f4f;
        color: white;
        padding: 0.25em 0.5em;
        border-radius: 0.5em;
        font-size: 0.9em;
    }
</style>
