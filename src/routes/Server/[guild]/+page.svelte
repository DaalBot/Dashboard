<script lang="ts">
    import { onMount } from 'svelte';
    import { page } from '$app/state';
    import { DBAPI, modules } from '@daalbot/api';

    let activity: Array<{
    method: string;
    route: string;
    ts: number;
    executor: string;
    comment: string;
}> = [];

    let basicGuildInfo: {
        id: string;
        name: string;
        icon: string | null;
        owner: boolean;
        permissions: number;
    } = {
        id: '',
        name: '',
        icon: null,
        owner: false,
        permissions: 0
    };

    onMount(async() => {
        const guildId = page.params.guild ?? 'THIS_IS_BUGGED';
        const api = new DBAPI({
            auth: {
                token: localStorage.getItem('accessToken') || '',
                type: 'User'
            },
            guildId
        });

        activity = await modules.getActivity(api);

        basicGuildInfo = JSON.parse(localStorage.getItem('user') ?? '').guilds.find((g: { id: string; }) => g.id === guildId);
        if (!basicGuildInfo) {
            alert('Guild not found - basic lookup failed');
            window.location.href = '/Error?type=failed';
        }
    })
</script>

<svelte:head>
    <title>Server - {basicGuildInfo.name}</title>
</svelte:head>

<h2>
    Modules
</h2>
<div class="modules">
    <h3>Automation</h3>
    <div class="group">
        <a href="/Server/{page.params.guild}/events">
            Events
        </a>
        <a href="/Server/{page.params.guild}/monitoring/role-invites"> <!-- This would probably be better under "Monitoring" but im trying to keep things even -->
            Role Invites
        </a>
    </div>
    <h3>XP</h3>
    <div class="group">
        <a href="/Server/{page.params.guild}/xp/users">
            Users & Leaderboard
        </a>
        <a href="/Server/{page.params.guild}/xp/rewards">
            Rewards
        </a>
    </div>
    <h3>Monitoring</h3>
    <div class="group">
        <a href="/Server/{page.params.guild}/monitoring/logs">
            Logs
        </a>
        <a href="/Server/{page.params.guild}/monitoring/invites">
            Invite Tracking
        </a>
    </div>
    <h3>Tools</h3>
    <div class="group">
        <a href="/Server/{page.params.guild}/tools/message-builder">
            Message Builder
        </a>
    </div>
</div>

<h2>
    Recent Activity
</h2>
<div class="activity">
    {#each activity as item}
        <div class="activity-item">
            <h3>{item.comment}</h3>
            <p class="executor">
                {#if item.executor != '-1'}
                    Automated (Guild Key)
                {:else}
                    <a href={`https://discordlookup.com/user/${item.executor}`} target="_blank">
                        Click to see executor
                    </a>
                {/if}
                <span class="timestamp">{new Date(item.ts).toLocaleString()}</span>
            </p>
        </div>
    {/each}
</div>

<style>
    .activity {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .activity-item {
        background-color: #3f3f3f;
        padding: 1rem;
        border-radius: 1rem;
    }

    .activity-item h3 {
        margin: 0;
        font-size: 1.2rem;
    }

    .activity-item p {
        margin: 0.5rem 0;
    }

    .activity-item span {
        font-size: 0.9rem;
        color: #555;
    }

    .activity-item .executor {
        font-weight: bold;
    }

    .activity-item .timestamp {
        font-size: 0.8rem;
        color: #999;
    }

    .modules {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .group {
        display: flex;
        flex-direction: row;
        gap: 1rem;
        width: 100%;
    }

    .group a {
        flex: 1;
        text-align: center;
        display: flex;
        flex-direction: row;
        gap: 1rem;
    }

    .group a {
        text-decoration: none;
        color: #fff;
        background-color: #3f3f3f;
        font-weight: bold;
        padding: 1.5rem 2rem;
        border-radius: 1rem;
        transition: background-color 0.3s ease;
    }

    .group a:hover {
        background-color: #4f4f4f;
    }

    .group a:active {
        background-color: #5f5f5f;
    }

    .modules h3 {
        margin: 0;
        font-size: 1.5rem;
        color: #fff;
        font-weight: bold;
        text-align: center;
        margin-bottom: 1rem;
        padding: .75rem;
        border-radius: 1rem;

        background-color: #826ae3;
    }
</style>