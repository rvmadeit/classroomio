<script lang="ts">
  import { onMount, createEventDispatcher } from 'svelte';
  import Tabs from '$lib/components/Tabs/index.svelte';
  import { getSupabase } from '$lib/utils/functions/supabase';
  import TabContent from '$lib/components/TabContent/index.svelte';
  import { snackbar } from '../Snackbar/store';
  import Modal from '$lib/components/Modal/index.svelte';
  import { handleOpenWidget } from '$lib/components/CourseLandingPage/store';
  import { queryUnsplash } from './utils';
  import PrimaryButton from '$lib/components/PrimaryButton/index.svelte';
  import { t } from '$lib/utils/functions/translations';

  export let imageURL = '';

  const supabase = getSupabase();
  const dispatch = createEventDispatcher();
  const tabs = [
    { label: 'Unsplash', value: 'unsplash' },
    { label: 'Upload', value: 'upload' }
  ];

  let isUploading = false;
  let currentTab = tabs[0].value;
  let searchQuery = '';
  let unsplashImages: {
    id: string | number;
    user: {
      name: string;
      username: string;
    };
    urls: {
      regular: string;
    };
    alt_description: string;
  }[] = [];
  let fileInput: HTMLInputElement;

  let label = $t('snackbar.landing_page_settings.error.label');

  const onChange = (tabValue: string | number) => () => (currentTab = `${tabValue}`);

  async function handleImageClick(img: string) {
    dispatch('change');
    imageURL = img;
    $handleOpenWidget.open = false;
  }

  const onFileSelected = () => {
    const file = fileInput?.files?.[0];
    const sizeInkb = file?.size! / 1024;
    if (sizeInkb > 500) {
      snackbar.error('snackbar.landing_page_settings.error.file_size');
      label = $t('snackbar.landing_page_settings.error.try_again');
      return;
    }
    if (file) {
      let reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onload = (e) => {
        uploadImage(file);
      };
    }
  };

  const uploadImage = async (image: File) => {
    isUploading = true;
    if (!image) {
      return;
    }
    const filename = `uploadwidget/${Date.now()}` + image.name;
    const { data } = await supabase.storage.from('avatars').upload(filename, image, {
      cacheControl: '3600',
      upsert: false
    });

    if (data) {
      const { data: response } = await supabase.storage.from('avatars').getPublicUrl(filename);
      imageURL = response.publicUrl;
      dispatch('change');
    }
    isUploading = false;

    snackbar.success(`snackbar.landing_page_settings.success.complete`);
    $handleOpenWidget.open = false;
  };

  function handleUpload() {
    fileInput.click();
  }

  async function handleSubmit() {
    try {
      unsplashImages = await queryUnsplash(searchQuery || 'rocks');
    } catch (error) {
      snackbar.error('snackbar.landing_page_settings.error.fetch_error');
      console.error('Error fetching images from Unsplash:', error);
    }
  }

  onMount(handleSubmit);
</script>

<Modal
  onClose={() => ($handleOpenWidget.open = false)}
  bind:open={$handleOpenWidget.open}
  width="w-3/5"
  maxWidth=""
  modalHeading={$t('course.navItem.landing_page.upload_widget.title')}
>
  <div class="w-full bg-white p-5 dark:bg-inherit">
    <Tabs {tabs} {currentTab} {onChange}>
      <slot:fragment slot="content">
        <TabContent value={tabs[1].value} index={currentTab}>
          <!-- Your Upload content here -->
          <div class="w-full">
            <input
              type="file"
              style="display: none;"
              bind:this={fileInput}
              on:change={onFileSelected}
              disabled={isUploading}
            />
            <PrimaryButton
              {label}
              onClick={handleUpload}
              isLoading={isUploading}
              className="w-full font-semibold m-auto"
            />
            <p class="my-2 text-center text-sm text-gray-500">
              {$t('course.navItem.landing_page.upload_widget.width')}
            </p>
            <p class="text-center text-sm text-gray-500">
              {$t('course.navItem.landing_page.upload_widget.size')}
            </p>
          </div>
        </TabContent>
        <TabContent value={tabs[0].value} index={currentTab}>
          <!-- Your Images content here -->
          <div class="h-full overflow-y-scroll">
            <form on:submit|preventDefault={handleSubmit} class="mt-1 flex gap-2 pb-3">
              <input
                type="text"
                bind:value={searchQuery}
                name=""
                id=""
                class="ml-2 w-[85%] rounded-lg dark:text-black"
              />
              <button
                type="submit"
                class="rounded-lg border-[1px] border-gray-500 bg-white px-3 py-1 text-black"
                >{$t('course.navItem.landing_page.upload_widget.submit')}</button
              >
            </form>
            {#if unsplashImages && unsplashImages.length > 0}
              <div
                class="hide-scrollbar flex max-h-[300px] flex-row flex-wrap items-center gap-2 px-[10px]"
              >
                {#each unsplashImages as unsplashImages (unsplashImages.id)}
                  <div>
                    <div class="relative h-[130px] w-[195px] overflow-hidden">
                      <button on:click={() => handleImageClick(unsplashImages.urls.regular)}>
                        <img
                          src={unsplashImages.urls.regular}
                          alt={unsplashImages.alt_description}
                          class="h-full w-full cursor-pointer rounded-md object-cover hover:opacity-80"
                        />
                      </button>
                    </div>
                    {#if unsplashImages.user.name}
                      <p class="mt-1 text-center text-xs font-light text-gray-500">
                        By <a
                          href={`https://unsplash.com/@${unsplashImages.user.username}`}
                          target="_blank"
                          class="hover:text-red-700">{unsplashImages.user.name}</a
                        >
                      </p>
                    {/if}
                  </div>
                {/each}
              </div>
            {:else}
              <p class="pt-7 text-center">
                {$t('course.navItem.landing_page.upload_widget.no_images')}
              </p>
            {/if}
          </div>
        </TabContent>
      </slot:fragment>
    </Tabs>
  </div>
</Modal>

<style>
  .hide-scrollbar {
    width: 100%;
    overflow: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;
  }
  .hide-scrollbar::-webkit-scrollbar {
    width: 0;
    background: transparent;
  }
</style>
