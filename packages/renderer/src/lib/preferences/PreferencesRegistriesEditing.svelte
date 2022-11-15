<script lang="ts">
import type { Registry } from '@tmpwip/extension-api';
import { onMount } from 'svelte';
import { registriesInfos } from '../../stores/registries';
import ListItemButtonIconMenu from '../ui/ListItemButtonIconMenu.svelte';
import { faTrash } from '@fortawesome/free-solid-svg-icons';
import { faUser } from '@fortawesome/free-solid-svg-icons';
import { faUserPen } from '@fortawesome/free-solid-svg-icons';

let modifiedRegistries: Registry[] = [];
let loginErrorMessages: { serverUrl: string; error: string }[] = [];
let showPasswordForServerUrls: string[] = [];
let showNewRegistryForm = false;
let providerSourceName: string;
let showMenuForServerUrl: string | undefined;
const newRegistryRequest = {
  source: '',
  serverUrl: '',
  username: '',
  secret: '',
} as Registry;
let outsideWindow;

onMount(async () => {
  let providerSourceNames = await window.getImageRegistryProviderNames();
  if (providerSourceNames.length > 0) {
    providerSourceName = providerSourceNames[0];
  }
});

function isRegistryModified(serverUrl: string): boolean {
  return modifiedRegistries.some(r => r.serverUrl === serverUrl);
}

function markRegistryAsModified(registry: Registry) {
  setPasswordForRegistryVisible(registry.serverUrl, false);

  const modifiedRegistry = {
    source: registry.source,
    serverUrl: registry.serverUrl,
    username: registry.username,
    secret: registry.secret,
  } as Registry;

  modifiedRegistries.push(modifiedRegistry);

  updateUI();
}

function markRegistryAsClean(serverUrl: string) {
  const index = modifiedRegistries.findIndex(r => r.serverUrl === serverUrl, 0);
  if (index > -1) {
    modifiedRegistries.splice(index, 1);
  }

  updateUI();
}

function getModifiedRegistry(serverUrl: string): Registry | undefined {
  return modifiedRegistries.find(r => r.serverUrl === serverUrl);
}

function updateRegistryUsername(event: InputEvent, serverUrl: string) {
  let registryToUpdate = getModifiedRegistry(serverUrl);

  if (registryToUpdate !== undefined) {
    registryToUpdate.username = (event.target as any).value;
  }
}

function updateRegistryPassword(event: InputEvent, serverUrl: string) {
  let registryToUpdate = getModifiedRegistry(serverUrl);

  if (registryToUpdate !== undefined) {
    registryToUpdate.secret = (event.target as any).value;
  }
}

function setPasswordForRegistryVisible(serverUrl: string, visible: boolean) {
  const index = showPasswordForServerUrls.findIndex(r => r === serverUrl);

  if (visible && index === -1) {
    showPasswordForServerUrls.push(serverUrl);
  } else if (!visible && index > -1) {
    showPasswordForServerUrls.splice(index, 1);
  }

  updateUI();
}

function isPasswordForRegistryVisible(serverUrl: string): boolean {
  const index = showPasswordForServerUrls.findIndex(r => r === serverUrl);

  return index > -1;
}

function isCredentialsEmpty(registry: Registry): boolean {
  return !registry.username && !registry.secret;
}

function getLoginErrorMessage(serverUrl: string): string {
  return loginErrorMessages.find(o => o.serverUrl === serverUrl)?.error || '';
}

function setLoginErrorMessage(serverUrl: string, message: string | undefined) {
  if (message) {
    loginErrorMessages.push({ serverUrl: serverUrl, error: message });
    updateUI();
  } else {
    const index = loginErrorMessages.findIndex(o => o.serverUrl === serverUrl, 0);
    if (index > -1) {
      loginErrorMessages.splice(index, 1);
    }
  }
}

function setNewRegistryFormVisible(visible: boolean) {
  showNewRegistryForm = visible;
}

async function loginToRegistry(registry: Registry) {
  let request = registry;

  setLoginErrorMessage(request.serverUrl, undefined);

  request.source = providerSourceName;

  const newRegistry = request === newRegistryRequest;

  if (!newRegistry) {
    request = getModifiedRegistry(request.serverUrl);

    if (request === undefined) {
      request = registry;
    }
  }

  try {
    if (newRegistry) {
      await window.createImageRegistry(request.source, request);
    } else {
      await window.updateImageRegistry(request);
    }
  } catch (error) {
    setLoginErrorMessage(request.serverUrl, error.message);
  }

  if (getLoginErrorMessage(request.serverUrl).length === 0) {
    if (newRegistry) {
      showNewRegistryForm = false;

      //reuse this object for future requests
      newRegistryRequest.serverUrl = '';
      newRegistryRequest.username = '';
      newRegistryRequest.secret = '';
    } else {
      markRegistryAsClean(request.serverUrl);
    }
  }
}

function removeExistingRegistry(registry: Registry) {
  window.unregisterImageRegistry(registry);
}

function isLoginButtonDisabled(r: Registry): boolean {
  const registry = r === newRegistryRequest ? newRegistryRequest : getModifiedRegistry(r.serverUrl);
  return !registry.serverUrl || !registry.username || !registry.secret;
}

async function setRegistryMenuVisible(event: PointerEvent, serverUrl: string | undefined) {
  if (showMenuForServerUrl === serverUrl) {
    closeRegistryMenu();
  } else {
    showMenuForServerUrl = serverUrl;
  }
  event.preventDefault();
  event.stopPropagation();
}

function closeRegistryMenu() {
  showMenuForServerUrl = undefined;
}

function updateUI() {
  registriesInfos.update(r => r);
}
function handleEscape({ key }) {
  if (key === 'Escape') {
    closeRegistryMenu();
  }
}

function onWindowClick(e) {
  if (outsideWindow.contains(e.target) == false) closeRegistryMenu();
}
</script>

<svelte:window on:keyup="{handleEscape}" on:click="{onWindowClick}" />

<div class="flex w-full flex-col p-2 bg-zinc-800">
  <h1 class="capitalize text-lg font-bold py-8 px-8">Registries</h1>
  <div class="container mx-auto">
    <!-- Registries table start -->
    <div class="w-full border-t border-b border-gray-600">
      <div class="flex w-full">
        <div class="flex-1 text-left py-4 pl-10 text-sm font-bold w-auto">Registry Location</div>
        <div class="text-left py-4 text-sm font-bold w-1/4">Username</div>
        <div class="text-left py-4 text-sm font-bold w-2/5">Password</div>
      </div>

      {#each $registriesInfos as registry}
        <!-- Registry row start -->
        <div class="flex flex-col w-full border-t border-gray-600">
          <div class="flex flex-row">
            <!-- Server URL -->
            <div class="flex-1 pt-5 pb-5 pl-12 text-sm w-auto">{registry.serverUrl}</div>

            <!-- Username -->
            <div class="pt-5 pb-5 text-sm w-1/4">
              {#if isRegistryModified(registry.serverUrl)}
                <div class="text-left h-7 pr-5 -mt-0.5 -mb-2 text-sm w-full">
                  <input
                    type="text"
                    placeholder="Username"
                    value="{getModifiedRegistry(registry.serverUrl).username}"
                    on:input="{event => updateRegistryUsername(event, registry.serverUrl)}"
                    class="block px-3 block w-full h-full transition ease-in-out delay-50 bg-zinc-900 text-gray-400 placeholder-gray-400 rounded-sm focus:outline-none" />
                </div>
              {:else if isCredentialsEmpty(registry)}
                <button class="font-bold" on:click="{() => markRegistryAsModified(registry)}">Login now</button>
              {:else}
                {registry.username}
              {/if}
            </div>

            <!-- Password -->
            <div class="pt-5 pb-5 text-sm w-full w-2/5">
              <div class="flex flex-row">
                {#if isRegistryModified(registry.serverUrl)}
                  <div class="text-left h-7 pr-5 -mt-0.5 -mb-2 text-sm w-full">
                    <input
                      type="text"
                      placeholder="Password"
                      value="{getModifiedRegistry(registry.serverUrl).secret}"
                      on:input="{event => updateRegistryPassword(event, registry.serverUrl)}"
                      class="block px-3 block w-full h-full transition ease-in-out delay-50 bg-zinc-900 text-gray-400 placeholder-gray-400 rounded-sm focus:outline-none" />
                  </div>
                  <div class="h-7 pr-5 -mt-0.5 -mb-2 text-sm w-28">
                    <button
                      on:click="{() => loginToRegistry(registry)}"
                      class="pf-c-button pf-m-primary transition ease-in-out delay-50 hover:cursor-pointer w-full h-full rounded-md shadow hover:shadow-lg"
                      type="button">
                      Login
                    </button>
                  </div>
                  <div class="h-7 pr-5 -mt-0.5 -mb-2 text-sm w-28">
                    <button
                      on:click="{() => markRegistryAsClean(registry.serverUrl)}"
                      class="transition ease-in-out delay-50 hover:cursor-pointer w-full h-full"
                      type="button">
                      Cancel
                    </button>
                  </div>
                {:else}
                  <!-- Password field start -->
                  <div class="container mx-auto w-full self-center items-center truncate">
                    {#if isCredentialsEmpty(registry)}
                      <span class="no-user-select">&nbsp;</span>
                    {:else if isPasswordForRegistryVisible(registry.serverUrl)}
                      {registry.secret}
                    {:else}
                      ····················
                    {/if}
                  </div>
                  <!-- Password field end -->
                  <!-- Show/hide password start -->
                  <div class="self-center w-8">
                    {#if !isCredentialsEmpty(registry)}
                      {#if isPasswordForRegistryVisible(registry.serverUrl)}
                        <button
                          type="button"
                          class="inline-flex w-full justify-center text-sm shadow-sm"
                          id="hide-password"
                          title="Hide password"
                          aria-expanded="true"
                          aria-haspopup="true"
                          on:click="{() => setPasswordForRegistryVisible(registry.serverUrl, false)}">
                          <i class="fa fa-eye-slash"></i>
                        </button>
                      {:else}
                        <button
                          type="button"
                          class="inline-flex w-full justify-center text-sm shadow-sm"
                          id="show-password"
                          title="Show password"
                          aria-expanded="true"
                          aria-haspopup="true"
                          on:click="{() => setPasswordForRegistryVisible(registry.serverUrl, true)}">
                          <i class="fa fa-eye"></i>
                        </button>
                      {/if}
                    {/if}
                  </div>
                  <!-- Show/hide password end -->
                  <!-- Registry menu start -->
                  <div class="relative self-center w-8 dropdown pr-20" bind:this="{outsideWindow}">
                    <div>
                      <button
                        on:click="{event => setRegistryMenuVisible(event, registry.serverUrl)}"
                        type="button"
                        class="inline-flex w-full justify-center text-sm shadow-sm w-6 h-4"
                        id="registry-menu"
                        aria-expanded="true"
                        aria-haspopup="true">
                        <i class="fa fa-ellipsis-v"></i>
                      </button>
                    </div>

                    {#if showMenuForServerUrl && showMenuForServerUrl === registry.serverUrl}
                      <div
                        id="dropdown"
                        class="origin-top-right absolute right-3/4 w-40 z-10 m-2 rounded-md shadow-lg bg-zinc-700 ring-1 ring-black ring-opacity-5 divide-y divide-gray-600 focus:outline-none overflow-hidden">
                        <ListItemButtonIconMenu
                          title="Login"
                          onClick="{() => markRegistryAsModified(registry)}"
                          hidden="{!isCredentialsEmpty(registry)}"
                          icon="{faUser}" />
                        <ListItemButtonIconMenu
                          title="Edit password"
                          onClick="{() => markRegistryAsModified(registry)}"
                          hidden="{isCredentialsEmpty(registry)}"
                          icon="{faUserPen}" />
                        <ListItemButtonIconMenu
                          title="Remove"
                          onClick="{() => removeExistingRegistry(registry)}"
                          icon="{faTrash}" />
                      </div>
                    {/if}
                  </div>
                  <!-- Registry menu end -->
                {/if}
              </div>
            </div>
          </div>
          <div class="flex flex-row-reverse w-full pb-3 -mt-2">
            <span class="w-2/3 pl-4 text-sm font-bold">
              {#if isRegistryModified(registry.serverUrl)}
                {getLoginErrorMessage(registry.serverUrl)}
              {/if}
            </span>
          </div>
        </div>
        <!-- Registry row end -->
      {/each}

      {#if showNewRegistryForm}
        <!-- Add new registry form start -->
        <div class="flex flex-col w-full border-t border-gray-600">
          <div class="flex flex-row w-full">
            <div class="flex-1 py-4 pr-5 pl-12 mt-0.5 text-sm w-auto">
              <input
                type="text"
                placeholder="registry.example.com*"
                bind:value="{newRegistryRequest.serverUrl}"
                class="px-2 block w-full h-7 transition ease-in-out delay-50 bg-zinc-900 text-gray-400 placeholder-gray-400 rounded-sm focus:outline-none" />
            </div>
            <div class="py-4 pr-5 mt-0.5 text-sm w-1/4">
              <input
                type="text"
                placeholder="Username"
                bind:value="{newRegistryRequest.username}"
                class="px-2 block w-full h-7 transition ease-in-out delay-50 bg-zinc-900 text-gray-400 placeholder-gray-400 rounded-sm focus:outline-none" />
            </div>
            <div class="py-4 pr-5 mt-0.5 text-sm w-2/5">
              <div class="flex flex-row h-7 space-x-5">
                <input
                  type="text"
                  placeholder="Password"
                  bind:value="{newRegistryRequest.secret}"
                  class="px-2 block w-full h-7 transition ease-in-out delay-50 bg-zinc-900 text-gray-400 placeholder-gray-400 rounded-sm focus:outline-none" />
                <button
                  on:click="{() => loginToRegistry(newRegistryRequest)}"
                  disabled="{isLoginButtonDisabled(newRegistryRequest)}"
                  class="inline pf-c-button pf-m-primary transition ease-in-out delay-50 hover:cursor-pointer w-28 h-full rounded-md shadow hover:shadow-lg justify-center"
                  type="button">
                  Login
                </button>
                <button
                  on:click="{() => setNewRegistryFormVisible(false)}"
                  class="transition ease-in-out delay-50 hover:cursor-pointer w-28 h-full justify-center"
                  type="button">
                  Cancel
                </button>
              </div>
            </div>
          </div>
          <div class="flex flex-row w-full pb-3 -mt-2 pl-12">
            <span class="text-sm font-bold whitespace-pre-line">
              {getLoginErrorMessage(newRegistryRequest.serverUrl)}
            </span>
          </div>
        </div>
        <!-- Add new registry form end -->
      {/if}
    </div>
    <!-- Registries table end -->
  </div>
  <!--{getLoginErrorMessage(newRegistryRequest)}-->

  <!-- Spacer start -->
  <div class="container h-full"></div>
  <!-- Spacer end -->

  <!-- Add new registry button start -->
  <div class="flex justify-end py-4 px-4 w-full">
    <button
      on:click="{() => setNewRegistryFormVisible(true)}"
      class="pf-c-button pf-m-primary transition ease-in-out delay-50 hover:cursor-pointer h-7 w-36 text-sm rounded-md shadow hover:shadow-lg"
      type="button"
      disabled="{showNewRegistryForm}">
      Add registry
    </button>
  </div>
  <!-- Add new registry button end -->
</div>
