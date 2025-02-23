<script lang="ts">
import { filtered, searchPattern } from '../stores/images';
import { onDestroy, onMount } from 'svelte';
import ImageEmptyScreen from './image/ImageEmptyScreen.svelte';

import { router } from 'tinro';
import type { ImageInfoUI } from './image/ImageInfoUI';
import ImageActions from './image/ImageActions.svelte';
import type { ImageInfo } from '../../../main/src/plugin/api/image-info';
import NoContainerEngineEmptyScreen from './image/NoContainerEngineEmptyScreen.svelte';
import { providerInfos } from '../stores/providers';
import PushImageModal from './image/PushImageModal.svelte';
import { ImageUtils } from './image/image-utils';
import NavPage from './ui/NavPage.svelte';
import ImageStatusIcon from './image/ImageStatusIcon.svelte';
import type { Unsubscriber } from 'svelte/store';
import { containersInfos } from '../stores/containers';
import type { ContainerInfo } from '../../../main/src/plugin/api/container-info';

let searchTerm = '';
$: searchPattern.set(searchTerm);

let images: ImageInfoUI[] = [];
let multipleEngines = false;

let pushImageModal = false;
let pushImageModalImageInfo = undefined;
function handlePushImageModal(imageInfo: ImageInfoUI) {
  pushImageModalImageInfo = imageInfo;
  pushImageModal = true;
}

$: providerConnections = $providerInfos
  .map(provider => provider.containerConnections)
  .flat()
  .filter(providerContainerConnection => providerContainerConnection.status === 'started');

// number of selected items in the list
$: selectedItemsNumber = images.filter(image => image.selected).length;

// do we need to unselect all checkboxes if we don't have all items being selected ?
$: selectedAllCheckboxes = images.filter(image => !image.inUse).every(image => image.selected);

let allChecked = false;

function updateImages() {
  const imageUtils = new ImageUtils();

  const computedImages = storeImages
    .map((imageInfo: ImageInfo) => imageUtils.getImagesInfoUI(imageInfo, storeContainers))
    .flat();

  // update selected items based on current selected items
  computedImages.forEach(image => {
    const matchingImage = images.find(
      currentImage => currentImage.id === image.id && currentImage.engineId === image.engineId,
    );
    if (matchingImage) {
      image.selected = matchingImage.selected;
    }
  });
  images = computedImages;

  // multiple engines ?
  const engineNamesArray = images.map(image => image.engineName);
  // remove duplicates
  const engineNames = [...new Set(engineNamesArray)];
  if (engineNames.length > 1) {
    multipleEngines = true;
  } else {
    multipleEngines = false;
  }
}

let imagessUnsubscribe: Unsubscriber;
let containersUnsubscribe: Unsubscriber;
let storeContainers: ContainerInfo[] = [];
let storeImages: ImageInfo[] = [];
onMount(async () => {
  containersUnsubscribe = containersInfos.subscribe(value => {
    storeContainers = value;
    updateImages();
  });

  imagessUnsubscribe = filtered.subscribe(value => {
    storeImages = value;
    updateImages();
  });
});

onDestroy(() => {
  // unsubscribe from the store
  if (imagessUnsubscribe) {
    imagessUnsubscribe();
  }
  if (containersUnsubscribe) {
    containersUnsubscribe();
  }
});

function closeModals() {
  pushImageModal = false;
}

function gotoBuildImage(): void {
  router.goto('/images/build');
}

function gotoPullImage(): void {
  router.goto('/images/pull');
}

function openDetailsImage(image: ImageInfoUI) {
  router.goto(`/images/${image.id}/${image.engineId}/${image.base64RepoTag}/summary`);
}

function toggleAllImages(value: boolean) {
  const toggleImages = images;
  // filter out all images used by a container
  toggleImages.filter(image => !image.inUse).forEach(image => (image.selected = value));
  images = toggleImages;
}

// delete the items selected in the list
let bulkDeleteInProgress = false;
async function deleteSelectedImages() {
  const selectedImages = images.filter(image => image.selected);

  if (selectedImages.length > 0) {
    bulkDeleteInProgress = true;
    await Promise.all(
      selectedImages.map(async image => {
        try {
          await window.deleteImage(image.engineId, image.id);
        } catch (e) {
          console.log('error while removing image', e);
        }
      }),
    );
    bulkDeleteInProgress = false;
  }
}
</script>

<NavPage
  bind:searchTerm
  title="images"
  subtitle="Hover over an image to view action buttons; click to open up full details.">
  <div slot="additional-actions" class="space-x-2 flex flex-nowrap">
    <button on:click="{() => gotoPullImage()}" class="pf-c-button pf-m-primary" type="button">
      <span class="pf-c-button__icon pf-m-start">
        <i class="fas fa-arrow-circle-down" aria-hidden="true"></i>
      </span>
      Pull Image
    </button>
    <button on:click="{() => gotoBuildImage()}" class="pf-c-button pf-m-primary" type="button">
      <span class="pf-c-button__icon pf-m-start">
        <i class="fas fa-cube" aria-hidden="true"></i>
      </span>
      Build Image
    </button>
  </div>

  <div slot="bottom-additional-actions" class="flex flex-row justify-start items-center w-full">
    {#if selectedItemsNumber > 0}
      <button
        class="pf-c-button pf-m-primary"
        on:click="{() => deleteSelectedImages()}"
        title="Delete {selectedItemsNumber} selected items"
        type="button">
        <span class="pf-c-button__icon pf-m-start">
          {#if bulkDeleteInProgress}
            <div class="mr-4">
              <i class="pf-c-button__progress">
                <span class="pf-c-spinner pf-m-md" role="progressbar">
                  <span class="pf-c-spinner__clipper"></span>
                  <span class="pf-c-spinner__lead-ball"></span>
                  <span class="pf-c-spinner__tail-ball"></span>
                </span>
              </i>
            </div>
          {:else}
            <i class="fas fa-trash" aria-hidden="true"></i>
          {/if}
        </span>
      </button>
      <span class="pl-2">On {selectedItemsNumber} selected items.</span>
    {/if}
  </div>

  <div class="min-w-full flex" slot="table">
    <table class="mx-5 w-full" class:hidden="{images.length === 0}">
      <!-- title -->
      <thead>
        <tr class="h-7 uppercase text-xs text-gray-500">
          <th class="whitespace-nowrap w-5"></th>
          <th class="px-2 w-5"
            ><input
              type="checkbox"
              indeterminate="{selectedItemsNumber > 0 && !selectedAllCheckboxes}"
              bind:checked="{allChecked}"
              on:click="{event => toggleAllImages(event.currentTarget.checked)}"
              class="cursor-pointer invert hue-rotate-[218deg] brightness-75" /></th>
          <th class="text-center font-extrabold w-10">status</th>
          <th class="text-center font-extrabold w-10">Name</th>
          <th class="text-center">Creation date</th>
          <th class="px-6 whitespace-nowrap text-end">size</th>
          <th class="text-center">actions</th>
        </tr>
      </thead>
      <tbody class="">
        {#each images as image}
          <tr class="group h-12 bg-zinc-900 hover:bg-zinc-700">
            <td class="rounded-tl-lg rounded-bl-lg w-5"> </td>
            <td class="px-2">
              <input
                type="checkbox"
                bind:checked="{image.selected}"
                disabled="{image.inUse}"
                class:cursor-pointer="{!image.inUse}"
                class:cursor-not-allowed="{image.inUse}"
                class:opacity-10="{image.inUse}"
                title="{image.inUse ? 'Image is used by a container' : ''}"
                class=" invert hue-rotate-[218deg] brightness-75 " />
            </td>
            <td class="bg-zinc-900 group-hover:bg-zinc-700 flex flex-row justify-center h-12">
              <ImageStatusIcon inUse="{image.inUse}" />
            </td>
            <td class="whitespace-nowrap  w-10 hover:cursor-pointer" on:click="{() => openDetailsImage(image)}">
              <div class="flex items-center">
                <div class="">
                  <div class="flex flex-row items-center">
                    <div class="text-sm text-gray-200">{image.name}</div>
                  </div>
                  <div class="flex flex-row items-center">
                    <div class="text-xs text-violet-400">{image.shortId}</div>
                    <div class="ml-1 text-xs font-extra-light text-gray-300">{image.tag}</div>
                  </div>
                  <div class="flex flex-row text-xs font-extra-light text-gray-500">
                    <!-- Hide in case of single engine-->
                    {#if multipleEngines}
                      <div class="px-2 inline-flex text-xs font-extralight rounded-full bg-slate-800 text-slate-400">
                        {image.engineName}
                      </div>
                    {/if}
                  </div>
                </div>
              </div>
            </td>
            <td class="px-6 py-2 whitespace-nowrap w-10">
              <div class="flex items-center">
                <div class="ml-2 text-sm text-gray-200">{image.humanCreationDate}</div>
              </div>
            </td>
            <td class="px-6 py-2 whitespace-nowrap w-10">
              <div class="flex">
                <div class="w-full text-right text-sm text-gray-200">{image.humanSize}</div>
              </div>
            </td>
            <td class="px-6 whitespace-nowrap rounded-tr-lg rounded-br-lg ">
              <div class="flex opacity-0 flex-row justify-end group-hover:opacity-100">
                <ImageActions image="{image}" onPushImage="{handlePushImageModal}" />
              </div>
            </td>
          </tr>
          <tr><td class="leading-[8px]">&nbsp;</td></tr>
        {/each}
      </tbody>
    </table>
  </div>
  <div slot="empty" class="min-h-full">
    {#if providerConnections.length > 0}
      <ImageEmptyScreen images="{$filtered}" />
    {:else}
      <NoContainerEngineEmptyScreen />
    {/if}

    {#if pushImageModal}
      <PushImageModal
        imageInfoToPush="{pushImageModalImageInfo}"
        closeCallback="{() => {
          closeModals();
        }}" />
    {/if}
  </div>
</NavPage>
