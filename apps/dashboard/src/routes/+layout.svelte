<script lang="ts">
  import { onMount } from 'svelte';
  import { injectSpeedInsights } from '@vercel/speed-insights/sveltekit';
  import { fly } from 'svelte/transition';
  import { derived } from 'svelte/store';
  import { goto } from '$app/navigation';
  import { browser } from '$app/environment';
  import { page, navigating } from '$app/stores';
  import isEmpty from 'lodash/isEmpty';
  import { Theme, ToastNotification, Loading } from 'carbon-components-svelte';
  import type { CarbonTheme } from 'carbon-components-svelte/types/Theme/Theme.svelte';
  import LandingNavigation from '$lib/components/Navigation/index.svelte';
  import OrgNavigation from '$lib/components/Navigation/app.svelte';
  import LMSNavigation from '$lib/components/Navigation/lms.svelte';
  import OrgLandingPage from '$lib/components/Org/LandingPage/index.svelte';
  import Snackbar from '$lib/components/Snackbar/index.svelte';
  import Backdrop from '$lib/components/Backdrop/index.svelte';
  import OrgSideBar from '$lib/components/Org/SideBar.svelte';
  import LMSSideBar from '$lib/components/LMS/SideBar.svelte';
  import Apps from '$lib/components/Apps/index.svelte';
  import PlayQuiz from '$lib/components/Org/Quiz/Play/index.svelte';
  import { course } from '$lib/components/Course/store';
  import { isCoursesPage, isOrgPage, isLMSPage, isQuizPage } from '$lib/utils/functions/app';
  import showAppsSideBar from '$lib/utils/functions/showAppsSideBar';
  import isPublicRoute from '$lib/utils/functions/routes/isPublicRoute';
  import { user, profile } from '$lib/utils/store/user';
  import { getSupabase, isSupabaseTokenInLocalStorage } from '$lib/utils/functions/supabase';
  import { isMobile } from '$lib/utils/store/useMobile';
  import { ROUTE } from '$lib/utils/constants/routes';
  import { getOrganizations } from '$lib/utils/services/org';
  import { toggleBodyByMode } from '$lib/utils/functions/app';
  import { appStore } from '$lib/utils/store/app';
  import { currentOrg } from '$lib/utils/store/org';
  import { setTheme } from '$lib/utils/functions/theme';
  import hideNavByRoute from '$lib/utils/functions/routes/hideNavByRoute';
  import shouldRedirectOnAuth from '$lib/utils/functions/routes/shouldRedirectOnAuth';
  import AddOrgModal from '$lib/components/Org/AddOrgModal/AddOrgModal.svelte';
  import { identifyPosthogUser, initPosthog } from '$lib/utils/services/posthog';
  import { initSentry, setSentryUser } from '$lib/utils/services/sentry';

  import '../app.postcss';

  export let data;

  let supabase = getSupabase();
  let path = $page.url?.pathname?.replace('/', '');
  let carbonTheme: CarbonTheme = 'white';

  const delayedPreloading = derived(navigating, (currentPreloading, set) => {
    setTimeout(() => set(currentPreloading), 250);
  });

  function setupAnalytics() {
    // Set up sentry
    initSentry();

    // Set up posthog
    initPosthog();
  }

  function setAnalyticsUser() {
    if ($profile.id) {
      setSentryUser({
        id: $profile.id,
        username: $profile.username,
        email: $profile.email,
        fullname: $profile.fullname
      });

      identifyPosthogUser($profile.id, { email: $profile.email, name: $profile.fullname });
    }
  }

  async function getProfile() {
    const params = new URLSearchParams(window.location.search);
    // Get user profile
    const {
      data: { session }
    } = await supabase.auth.getSession();
    const { user: authUser } = session || {};
    console.log('Get user', authUser);

    if (!authUser && !isPublicRoute($page.url?.pathname)) {
      return goto('/login?redirect=/' + path);
    }

    // Check if user has profile
    let {
      data: profileData,
      error,
      status
    } = await supabase.from('profile').select(`*`).eq('id', authUser?.id).single();
    console.log('Get profile', profileData);

    if (error && !profileData && status === 406 && authUser) {
      // User wasn't found, create profile
      console.log(`User wasn't found, create profile`);

      const [regexUsernameMatch] = [...(authUser.email?.matchAll(/(.*)@/g) || [])];

      const { data: newProfileData, error } = await supabase
        .from('profile')
        .insert({
          id: authUser.id,
          username: regexUsernameMatch[1] + `${new Date().getTime()}`,
          fullname: regexUsernameMatch[1],
          email: authUser.email
        })
        .select();

      // Profile created, go to onboarding or lms
      if (!error && newProfileData) {
        $user.fetchingUser = false;
        $user.isLoggedIn = true;
        $user.currentSession = authUser;

        profile.set(newProfileData[0]);

        setAnalyticsUser();

        if (data.isOrgSite) {
          const { data, error } = await supabase
            .from('organizationmember')
            .insert({
              organization_id: $currentOrg.id,
              profile_id: $profile.id,
              role_id: 3
            })
            .select();
          if (error) {
            console.error('Error adding user to organisation', error);
          } else {
            console.log('Success adding user to organisation', data);
            const memberId = data?.[0]?.id || '';

            $currentOrg.memberId = memberId;
          }

          if (params.get('redirect')) {
            goto(params.get('redirect') || '');
          } else {
            goto('/lms');
          }
          return;
        }

        // On invite page, don't go to onboarding
        if (!path.includes('invite')) {
          goto(ROUTE.ONBOARDING);
        }
      }
      $user.fetchingUser = false;
    } else if (profileData) {
      // Profile exists, go to profile page
      $user.fetchingUser = false;
      $user.isLoggedIn = true;
      $user.currentSession = authUser;
      profile.set(profileData);

      // Set user in sentry
      setAnalyticsUser();

      const orgRes = await getOrganizations(profileData.id);

      // student redirect
      if (data.isOrgSite) {
        if (params.has('redirect')) {
          goto(params.get('redirect') || '');
        } else if (shouldRedirectOnAuth(path)) {
          goto('/lms');
        }
      } else {
        // Not on invite page or no org, go to onboarding
        if (isEmpty(orgRes.orgs) && !path.includes('invite')) {
          goto(ROUTE.ONBOARDING);
        } else if (params.has('redirect')) {
          goto(params.get('redirect') || '');
        } else if (shouldRedirectOnAuth(path)) {
          // By default redirect to first organization
          goto(`/org/${orgRes.currentOrg.siteName}`);
        }
      }
      if (typeof orgRes?.currentOrg?.theme === 'string') {
        setTheme(orgRes.currentOrg.theme);
      }
    }

    if (!profileData && !isPublicRoute($page.url?.pathname)) {
      goto('/login?redirect=/' + path);
    }
  }

  function handleResize() {
    isMobile.update(() => window.innerWidth <= 760);
  }

  onMount(() => {
    console.log(
      'Welcome to ClassroomIO, we are grateful you chose us.',
      $page.url.host,
      `\nIs student domain: ${data.isOrgSite}`
    );

    if (browser) {
      // Update theme - dark or light mode
      $appStore.isDark = localStorage.getItem('mode') === 'dark';
      toggleBodyByMode($appStore.isDark);

      const theme = localStorage.getItem('theme');
      if (data.isOrgSite && data.org?.theme) {
        setTheme(data.org?.theme);
      } else if (theme) {
        setTheme(theme);
      }
    }

    setupAnalytics();

    injectSpeedInsights();

    handleResize();

    if (!isSupabaseTokenInLocalStorage() && !isPublicRoute($page.url?.pathname)) {
      console.log('No auth token and is not a public route, redirect to login', path);
      return goto('/login?redirect=/' + path);
    }

    const { data: authListener } = supabase.auth.onAuthStateChange((event, session) => {
      // Log key events
      console.log(`event`, event);
      if (event == 'PASSWORD_RECOVERY') {
        console.log('PASSWORD RESET');
      }

      if (path.includes('reset')) {
        console.log('Dont change auth when on reset page');
        return;
      }

      // Skip Authentication
      if (data.skipAuth) return;

      // Authentication Steps
      if (event === 'SIGNED_IN' || event === 'INITIAL_SESSION') {
        $user.fetchingUser = true;
        getProfile();
      } else if (!['TOKEN_REFRESHED'].includes(event)) {
        console.log('not logged in, go to login');
        return goto('/login');
      }
    });

    if (data.isOrgSite) {
      if (!data.org) {
        goto('/404');
      } else {
        $appStore.orgSiteName = data.orgSiteName;
        $appStore.isOrgSite = data.isOrgSite;

        currentOrg.set(data.org);
      }
    }

    return () => {
      console.log('unsubscribed');
      authListener.subscription.unsubscribe();
    };
  });

  $: path = $page.url?.pathname?.replace('/', '');
  $: carbonTheme = $appStore.isDark ? 'g100' : 'white';
</script>

<svelte:window on:resize={handleResize} />

<svelte:head>
  <link href="/css/carbon.css" rel="stylesheet" />
</svelte:head>

<Theme bind:theme={carbonTheme} />

<Snackbar />

{#if data.skipAuth}
  <PlayQuiz />
{:else if data.isOrgSite && !path}
  <OrgLandingPage orgSiteName={data.orgSiteName} org={data.org} />
{:else}
  <main class="dark:bg-black">
    {#if $navigating && $delayedPreloading}
      <Backdrop disableCenteredContent={true} className="opacity-90">
        <div class="h-full w-full relative" transition:fly={{ x: -200, duration: 500 }}>
          <ToastNotification kind="info-square" class="absolute bottom-5 left-5">
            <span slot="title" class="flex items-center">
              <span class="mr-2">Redirecting</span>
              <Loading withOverlay={false} small />
            </span>
            <span slot="caption">Taking you to the next page, please wait.</span>
          </ToastNotification>
        </div>
      </Backdrop>
    {/if}
    {#if !hideNavByRoute($page.url?.pathname)}
      {#if isOrgPage($page.url?.pathname) || $page.url?.pathname.includes('profile') || isCoursesPage(path)}
        <OrgNavigation bind:title={$course.title} isCoursePage={isCoursesPage(path)} />
      {:else if isLMSPage($page.url?.pathname)}
        <LMSNavigation />
      {:else}
        <LandingNavigation
          isOrgSite={data.isOrgSite}
          logo={data.isOrgSite ? $currentOrg.avatar_url : undefined}
          orgName={data.isOrgSite ? $currentOrg.name : undefined}
          disableSignup={false}
        />
      {/if}
    {/if}

    <div class="flex justify-between">
      {#if isOrgPage($page.url?.pathname)}
        <AddOrgModal />
        <div class="org-root w-full flex items-center justify-between">
          {#if !isQuizPage($page.url?.pathname)}
            <OrgSideBar />
          {/if}
          <div class="org-slot bg-white dark:bg-black w-full">
            <slot />
          </div>
        </div>
      {:else if isLMSPage($page.url?.pathname)}
        <div class="org-root w-full flex items-center justify-between">
          <LMSSideBar />
          <div class="org-slot bg-white dark:bg-black w-full">
            <slot />
          </div>
        </div>
      {:else}
        <slot />
      {/if}

      {#if showAppsSideBar(path)}
        <Apps />
      {/if}
    </div>
  </main>
{/if}

<style>
  main {
    background-color: white;
    box-sizing: border-box;
  }

  .org-root {
    height: calc(100vh - 48px);
  }
  .org-slot {
    min-width: calc(100vw - 250px);
    height: calc(100vh - 48px);
    overflow-y: auto;
  }

  :global(a:hover) {
    text-decoration: underline;
  }
  :global(:root) {
    --main-primary-color: rgba(29, 78, 216, 1);
    --border-color: #eaecef;
    --app-background-color: #fafbfc;
    --app-background: radial-gradient(
      circle at 10% 20%,
      rgb(239, 246, 249) 0%,
      rgb(206, 239, 253) 90%
    );
    --dark-app-background: radial-gradient(circle at 10% 20%, rgb(0 0 0) 0%, rgb(27 60 74) 90%);
  }

  :global(.app-background) {
    background: var(--app-background);
  }
  :global(.dark .app-background) {
    background: var(--dark-app-background);
  }

  :global(.bx--data-table-container) {
    width: 100%;
  }

  :global(.dark svg.dark) {
    fill: #fff;
  }

  :global(.border-c) {
    border: 1px solid var(--border-color);
  }

  :global(.border-bottom-c) {
    border-bottom: 1px solid var(--border-color);
  }

  :global(.bx--search-close > svg) {
    fill: black;
  }

  :global(.dark .bx--search-close:hover > svg) {
    fill: #fff;
  }

  :global(.plyr__controls) {
    background:
      url(/logo-192.png) 99% 70% no-repeat,
      linear-gradient(rgba(0, 0, 0, 0), rgba(0, 0, 0, 0.5)) !important;
    background-size:
      50px auto,
      auto !important;
  }

  :global(.cards-container) {
    width: 90%;
    margin: 0 auto;
    padding: 0;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    column-gap: 12px;
    row-gap: 12px;
  }

  @media screen and (max-width: 768px) {
    :global(.cards-container) {
      width: 95%;
      margin: 0 auto;
      padding: 0;
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
      column-gap: 12px;
      row-gap: 12px;
    }
  }
  @media screen and (max-width: 457px) {
    .org-slot {
      min-width: 100%;
      max-height: calc(100vh - 48px);
      overflow-y: auto;
    }
  }
</style>