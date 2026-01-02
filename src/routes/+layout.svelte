<script lang="ts">
    import '@shoelace-style/shoelace/dist/components/visually-hidden/visually-hidden.js';
    import { onMount } from 'svelte';
    import { page } from '$app/state';
    import { state } from '$lib/state.svelte';
    import './styles.css';
    import config from '$lib/config.json';
    import type { LocalUserData } from '$lib/types';
    import { DBAPI } from '@daalbot/api';
    import { getMutualGuilds } from '@daalbot/api/dist/src/modules/currentUser';
    import { browser } from '$app/environment';
    
    let isMobile = false;
    let mobileShowMenu = false;
    let rightMenuExpanded = false;

    let mutualGuilds: string[] = [];
    let userData: LocalUserData = {
        guilds: [],
        id: "",
        username: "",
        discriminator: "",
        global_name: null,
        avatar: null
    };

    let isGuildPage = false;
    
    onMount(() => {
        if (page.url.pathname.startsWith('/Server/')) isGuildPage = true;
        else isGuildPage = false;

        const handleResize = () => {
            isMobile = window.innerWidth < 768;
        };
    
        handleResize();
        window.addEventListener('resize', handleResize);

        // If the user is logged out and isnt on the api page, redirect to discord login
        if (!localStorage.getItem('accessToken') && !page.url.pathname.includes('/api')) {
            if (import.meta.env.DEV) {
                console.log('DEV: Redirecting to dev_authURL');
                window.location.href = config.dev_authURL;
                return;
            }
            window.location.href = config.authURL;
        }

        const api = new DBAPI({
            auth: {
                token: localStorage.getItem('accessToken') || '',
                type: 'User'
            },
            guildId: '0' // Required but not required for the routes we need here
        });

        userData = localStorage.getItem('user') ? JSON.parse(localStorage.getItem('user') ?? '') : null;
        if (!userData) {
            console.warn('User not found - localStorage.getItem failed');
        }

        userData.guilds = userData.guilds.filter(g => parseInt(g.permissions) & 0x20); // Filter out guilds that the user does not have the MANAGE_GUILD permission

        if (sessionStorage.getItem('mutualGuilds')) {
            mutualGuilds = JSON.parse(sessionStorage.getItem('mutualGuilds') || '[]');
        } else {
            getMutualGuilds(api).then(guilds => {
                mutualGuilds = guilds;
                sessionStorage.setItem('mutualGuilds', JSON.stringify(guilds));
            }).catch(error => {
                console.error('Error fetching mutual guilds:', error);
                alert('Failed to fetch mutual guilds. You are about to be logged out and redirected to the login page to reauthenticate.');
                localStorage.removeItem('accessToken');
                localStorage.removeItem('user');
                sessionStorage.removeItem('guild');
                window.location.href = '/'; // Trigger login flow
            });
        }
    
        return () => {
            window.removeEventListener('resize', handleResize);
        };
    });
</script>

<div id="app">
    <header>
        {#if !isMobile}
        <div class="left">
            <button on:click={() => {
                if (!browser) return;
                window.location.href = '/';
            }}>
                Home
            </button>
        </div>
        <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
        <div class="center" style="{isGuildPage ? `background-image: url('https://cdn.discordapp.com/icons/${state.guild?.id}/${state.guild?.userData.icon}.png?size=256');` : ''}" on:click={() => {
            if (!browser || !isGuildPage) return;
        }} on:click={() => {
            if (!browser) return;
            if (isGuildPage) window.location.href = `/Server/${state.guild?.id}`;
        }}>
            {#if page.url.pathname === '/'}
                <h1>No server selected</h1>
                <p>Please select a server to manage</p>
            {:else if isGuildPage}
                {#if state.guild}
                    <h1 unselectable="on">{state.guild.userData.name}</h1>
                {:else}
                    <h1 unselectable="on">Loading...</h1>
                {/if}
                <sl-visually-hidden>
                    <a href={`/Server/${state.guild?.id}`}>Manage {state.guild?.userData.name}</a>
                </sl-visually-hidden>
            {:else}
                <h1>Unknown page</h1>
            {/if}
        </div>
        <div class="right">
            {#if rightMenuExpanded}
                <button on:click={() => {
                    if (!browser) return;
                    localStorage.removeItem('accessToken');
                    localStorage.removeItem('user');
                    sessionStorage.removeItem('guild');
                    window.location.href = '/'; // Trigger login flow
                }}>
                    Logout
                </button>
            {/if}
            <button style="display: flex; align-items: center; gap: 0.5rem;" on:click={() => rightMenuExpanded = !rightMenuExpanded}>
                {#if userData.avatar}
                    <img src="https://cdn.discordapp.com/avatars/{userData.id}/{userData.avatar}.png?size=32" alt="User Avatar" />
                {:else}
                    <img src="https://cdn.discordapp.com/embed/avatars/0.png" alt="Default Avatar" height="32" width="32" />
                {/if}
                {userData.username}
            </button>
        </div>
        {:else}
            <button style="width: 100%" on:click={() => mobileShowMenu = !mobileShowMenu}>
                Toggle Menu
            </button>
            <div class="mobile-menu" class:show={mobileShowMenu}>
                <button on:click={() => {
                    if (!browser) return;
                    localStorage.removeItem('accessToken');
                    localStorage.removeItem('user');
                    sessionStorage.removeItem('guild');
                    window.location.href = '/'; // Trigger login flow
                }}>
                    Logout
                </button>
                <button on:click={() => {
                    if (!browser) return;
                    window.location.href = '/';
                }}>
                    Home
                </button>
                {#if isGuildPage}
                <button on:click={() => {
                    if (!browser) return;
                    window.location.href = `/Server/${state.guild?.id}`;
                }}>
                    Manage {state.guild?.userData.name}
                </button>
                {/if}
            </div>
        {/if}
    </header>
    <main>
        <slot />
    </main>
</div>

<svelte:head>
    <title>{isGuildPage ? `Manage ${state.guild?.userData.name}` : 'Unknown Page'}</title>
</svelte:head>

<style>
    header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 1.25rem;
        background-color: #2f2f2f;
        border-radius: 1rem;
        box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
        color: white;

        margin-bottom: .75rem;
    }

    header button {
        background-color: #4a4a4a;
        color: white;
        border: none;
        padding: 0.75rem 1rem;
        border-radius: 0.5rem;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }
    
    header h1, header p {
        margin: 0;
    }

    header .center {
        background-color: #4e4e4e;
        padding: 0.5rem 1.5rem;
        border-radius: 0.5rem;
    }

    main {
        padding: 1.25rem;
        background-color: #1e1e1e;
        border-radius: 1rem;
        box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
        color: white;
    }

    .center {
        background-repeat: no-repeat;
        background-size: cover;
        background-position: center;
        position: relative;

        cursor: pointer;
    }

    .center::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: rgba(0, 0, 0, 0.5); /* Black overlay with 50% opacity */
        border-radius: 0.5rem;
        z-index: 0;
    }

    .center h1, .center p {
        position: relative;
        z-index: 1;
    }

    .right {
        display: flex;
        flex-direction: row;
        align-items: center;
        gap: 1rem;
    }
    
    #app {
        height: 100%;
    }

    @media (max-width: 768px) {
        header {
            flex-direction: column;
            align-items: flex-start;
        }

        header .center {
            width: 100%;
            padding: 0;
        }

        .mobile-menu {
            display: flex;
            flex-direction: column;
            width: 91%;
            background-color: #2f2f2f;
            border-radius: 0.5rem;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            gap: 0.5rem;
            
            /* Animation properties */
            max-height: 0;
            overflow: hidden;
            opacity: 0;
            padding: 0;
            margin: 0;
            transition: all 0.3s ease-in-out;
        }

        .mobile-menu.show {
            max-height: 200px;
            opacity: 1;
            padding: 1rem;
        }
    }
</style>