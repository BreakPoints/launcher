<script lang="ts">
  import { appWindow } from "@tauri-apps/api/window";
  import logo from "$assets/images/icon.webp";
  import { onMount } from "svelte";
  import { getVersion } from "@tauri-apps/api/app";
  import { Link } from "svelte-navigator";
  import Icon from "@iconify/svelte";
  import { UpdateStore } from "$lib/stores/AppStore";
  import { checkUpdate } from "@tauri-apps/api/updater";
  import { isInDebugMode } from "$lib/utils/common";
  import {
    getActiveVersion,
    getActiveVersionFolder,
    listDownloadedVersions,
  } from "$lib/rpc/versions";
  import { getLatestOfficialRelease } from "$lib/utils/github";
  import { VersionStore } from "$lib/stores/VersionStore";
  import { exceptionLog, infoLog } from "$lib/rpc/logging";
  import { _ } from "svelte-i18n";

  let launcherVerison = null;

  onMount(async () => {
    // Get current versions
    launcherVerison = `v${await getVersion()}`;

    $VersionStore.activeVersionType = await getActiveVersionFolder();
    $VersionStore.activeVersionName = await getActiveVersion();

    // Check for a launcher update
    // NOTE - the following code (checkUpdate) won't work unless you have `update` configuration
    // added to the tauri.conf.json
    if (!isInDebugMode()) {
      const updateResult = await checkUpdate();
      if (updateResult.shouldUpdate) {
        // TODO - store methods to clean this up
        let changeLog = [];
        try {
          changeLog = JSON.parse(updateResult.manifest.body);
        } catch (e) {
          exceptionLog(
            `Could not parse changelog JSON from release metadata - ${JSON.stringify(
              updateResult
            )}`,
            e
          );
        }
        $UpdateStore.launcher = {
          updateAvailable: true,
          versionNumber: updateResult.manifest.version,
          date: updateResult.manifest.date,
          changeLog: changeLog,
        };
        infoLog(`Launcher Update Available`);
      } else {
        $UpdateStore.launcher = {
          updateAvailable: false,
          versionNumber: null,
          date: null,
          changeLog: [],
        };
        infoLog(`Launcher is up to date - ${JSON.stringify(updateResult)}`);
      }
    }

    await checkIfLatestVersionInstalled();
  });

  async function checkIfLatestVersionInstalled() {
    // Check for an update to the tooling (right now, only if it's official)
    if (
      $VersionStore.activeVersionType === null ||
      $VersionStore.activeVersionType === "official"
    ) {
      const latestToolingVersion = await getLatestOfficialRelease();
      if ($VersionStore.activeVersionName !== latestToolingVersion.version) {
        // Check that we havn't already downloaded it
        let alreadyHaveRelease = false;
        const downloadedOfficialVersions = await listDownloadedVersions(
          "official"
        );
        for (const releaseVersion of downloadedOfficialVersions) {
          if (releaseVersion === latestToolingVersion.version) {
            alreadyHaveRelease = true;
            break;
          }
        }
        if (!alreadyHaveRelease) {
          $UpdateStore.selectedTooling = {
            updateAvailable: true,
            versionNumber: latestToolingVersion.version,
          };
        }
      }
    }
  }
</script>

<header
  class="flex flex-row basis-1/10 bg-[#101010] pl-2 pr-4 pt-1 pb-1 items-center z-10"
  data-tauri-drag-region
>
  <div class="flex flex-row items-center space-x-2 pointer-events-none">
    <img class="h-8" src={logo} aria-label="" />
    <p class="font-black text-white tracking-tight text-lg">OpenGOAL</p>
  </div>

  <div class="border-l border-[#9f9f9f] h-8 m-2" />

  <div class="flex flex-col text-neutral-500 mr-2 pointer-events-none">
    <p class="font-mono text-sm">Launcher</p>
    <p class="font-mono text-sm">Tooling</p>
  </div>

  <div class="flex flex-col text-neutral-300 mr-2 pointer-events-none">
    <p class="font-mono text-sm">
      {launcherVerison === null ? "not set!" : launcherVerison}
    </p>
    <p class="font-mono text-sm">
      {$VersionStore.activeVersionName === null
        ? "not set!"
        : $VersionStore.activeVersionName}
      {#if $VersionStore.activeVersionType === "unofficial"}
        (unf)
      {:else if $VersionStore.activeVersionType === "devel"}
        (dev)
      {/if}
    </p>
  </div>

  <div class="flex flex-col text-orange-500 pointer-events-none">
    <p
      class="font-mono text-sm hover:text-orange-300 {$UpdateStore.launcher
        .updateAvailable
        ? 'pointer-events-auto'
        : 'invisible pointer-events-none'}"
    >
      <Link class="font-mono" to="/update"
        >>&nbsp;{$_("header_updateAvailable")}</Link
      >
    </p>
    <p
      class="font-mono text-sm hover:text-orange-300 {$UpdateStore
        .selectedTooling.updateAvailable
        ? 'pointer-events-auto'
        : 'invisible pointer-events-none'}"
    >
      <Link class="font-mono " to="/settings/versions"
        >>&nbsp;{$_("header_updateAvailable")}</Link
      >
    </p>
  </div>

  <div class="flex space-x-4 text-xl ml-auto">
    <button class="hover:text-amber-600" on:click={() => appWindow.minimize()}>
      <Icon icon="material-symbols:chrome-minimize" width="24" height="24" />
    </button>
    <button class="hover:text-red-600" on:click={() => appWindow.close()}>
      <Icon icon="ic:baseline-close" width="24" height="24" />
    </button>
  </div>
</header>
