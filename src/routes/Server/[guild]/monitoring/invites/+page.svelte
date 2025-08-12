<script lang="ts">
    import { page } from "$app/state";
    import Slider from "$lib/components/Slider.svelte";
    import { DBAPI, modules } from "@daalbot/api";
    import type { CurrentGuildResponse } from "@daalbot/api/dist/src/modules/currentGuild";
    import type { InviteTrackingData } from "@daalbot/api/dist/src/modules/inviteTracking";
    import { onMount } from "svelte";

    let api: DBAPI;
    let guild: CurrentGuildResponse;

    let inviteTrackingState: InviteTrackingData = { enabled: false, invites: [] };
    let inviteTrackingLeaderboard: Record<string, { invites: number, codes: string[] }> = {};

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

            inviteTrackingState.invites.forEach(invite => {
                if (!inviteTrackingLeaderboard[invite.creator]) {
                    inviteTrackingLeaderboard[invite.creator] = { invites: 0, codes: [] };
                }
                inviteTrackingLeaderboard[invite.creator].invites += invite.uses;
                inviteTrackingLeaderboard[invite.creator].codes.push(invite.code);
            });
        } catch (error) {
            console.error("Failed to fetch invite tracking state:", error);
        }
    });
</script>

<h1>Invite Tracking</h1>
<h2>Should invite tracking be enabled?</h2>
<Slider bind:checked={inviteTrackingState.enabled} onChange={async() => {
    try {
        if (!inviteTrackingState.enabled) {
            const roleInviteStatus = await modules.getRoleLinks(api);
        
            if (roleInviteStatus.enabled) {
                const proceed = confirm(
                    `The role invite links module is enabled. Disabling invite tracking will prevent role invite links from functioning. Do you want to proceed?`
                );
                if (!proceed) {
                    inviteTrackingState.enabled = true; // revert the change
                    return;
                }
            }
        }

        await modules.toggleInviteTracking(api, inviteTrackingState.enabled);
    } catch (e) {
        console.error("Failed to toggle invite tracking:", e);
        alert("Failed to toggle invite tracking. Please try again later.");
    }
}} />

<h2>Leaderboard</h2>
<table>
    <thead>
        <tr>
            <th>User</th>
            <th>Invites</th>
            <th>Codes</th>
        </tr>
    </thead>
    <tbody>
        {#if Object.keys(inviteTrackingLeaderboard).length > 0}
            {#each Object.entries(inviteTrackingLeaderboard).sort((a, b) => b[1].invites - a[1].invites) as [userId, data]}
                <tr>
                    <td>{guild.members.find(user => user.userId == userId)?.displayName || `Unknown (${userId})`}</td>
                    <td>{data.invites}</td>
                    <td style="overflow-x: auto;">{data.codes.join(", ")}</td>
                </tr>
            {/each}
        {:else}
            <tr>
                <td colspan="3">No invites found.</td>
            </tr>
        {/if}
    </tbody>
</table>

<h2>Invites</h2>
<table>
    <thead>
        <tr>
            <th>Code</th>
            <th>Uses</th>
            <th>Users</th>
        </tr>
    </thead>
    <tbody>
        {#if inviteTrackingState.invites.length > 0}
        {#each inviteTrackingState.invites.sort((a, b) => b.uses - a.uses) as invite}
            <tr>
                <td>{invite.code}</td>
                <td>{invite.uses}</td>
                <td style="overflow-x: auto;">{invite.users.map(userId => guild.members.find(user => user.userId == userId)?.displayName).join(", ")}</td>
            </tr>
        {/each}
        {:else}
            <tr>
                <td colspan="3">No invites found, if you just enabled the feature please wait for invites to sync then reload.</td>
            </tr>
        {/if}
    </tbody>
</table>

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

    th {
        background-color: #3f3f3f;
    }
</style>
