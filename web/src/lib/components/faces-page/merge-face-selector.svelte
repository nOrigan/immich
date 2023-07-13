<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte';
  import { api, type PersonResponseDto } from '@api';
  import FaceThumbnail from './face-thumbnail.svelte';
  import { quintOut } from 'svelte/easing';
  import { fly } from 'svelte/transition';
  import ControlAppBar from '../shared-components/control-app-bar.svelte';
  import Button from '../elements/buttons/button.svelte';
  import Merge from 'svelte-material-icons/Merge.svelte';
  import CallMerge from 'svelte-material-icons/CallMerge.svelte';
  import { flip } from 'svelte/animate';
  import { NotificationType, notificationController } from '../shared-components/notification/notification';
  import ConfirmDialogue from '../shared-components/confirm-dialogue.svelte';
  import { handleError } from '$lib/utils/handle-error';
  import { invalidateAll } from '$app/navigation';

  export let person: PersonResponseDto;
  let people: PersonResponseDto[] = [];
  let selectedPeople: PersonResponseDto[] = [];
  let screenHeight: number;
  let isShowConfirmation = false;
  let dispatch = createEventDispatcher();

  $: hasSelection = selectedPeople.length > 0;
  $: unselectedPeople = people.filter((source) => !selectedPeople.includes(source) && source.id !== person.id);

  onMount(async () => {
    const { data } = await api.personApi.getAllPeople();
    people = data;
  });

  const onClose = () => {
    dispatch('go-back');
  };

  const onSelect = (selected: PersonResponseDto) => {
    if (selectedPeople.includes(selected)) {
      selectedPeople = selectedPeople.filter((person) => person.id !== selected.id);
      return;
    }

    if (selectedPeople.length >= 5) {
      notificationController.show({
        message: 'You can only merge up to 5 faces at a time',
        type: NotificationType.Info,
      });
      return;
    }

    selectedPeople = [selected, ...selectedPeople];
  };

  const handleMerge = async () => {
    try {
      const { data: results } = await api.personApi.mergePerson({
        id: person.id,
        mergePersonDto: { ids: selectedPeople.map(({ id }) => id) },
      });
      const count = results.filter(({ success }) => success).length;
      notificationController.show({
        message: `Merged ${count} ${count === 1 ? 'person' : 'people'}`,
        type: NotificationType.Info,
      });
      await invalidateAll();
      onClose();
    } catch (error) {
      handleError(error, 'Cannot merge faces');
    } finally {
      isShowConfirmation = false;
    }
  };
</script>

<svelte:window bind:innerHeight={screenHeight} />

<section
  transition:fly={{ y: 500, duration: 100, easing: quintOut }}
  class="absolute top-0 left-0 w-full h-full bg-immich-bg dark:bg-immich-dark-bg z-[9999]"
>
  <ControlAppBar on:close-button-click={onClose}>
    <svelte:fragment slot="leading">
      {#if hasSelection}
        Selected {selectedPeople.length}
      {:else}
        Merge faces
      {/if}
      <div />
    </svelte:fragment>
    <svelte:fragment slot="trailing">
      <Button
        size={'sm'}
        disabled={!hasSelection}
        on:click={() => {
          isShowConfirmation = true;
        }}
      >
        <Merge size={18} />
        <span class="ml-2"> Merge</span></Button
      >
    </svelte:fragment>
  </ControlAppBar>
  <section class="pt-[100px] px-[70px] bg-immich-bg dark:bg-immich-dark-bg">
    <section id="merge-face-selector relative">
      <div class="place-items-center place-content-center mb-10 h-[200px]">
        <p class="uppercase mb-4 dark:text-white text-center">Choose matching faces to merge</p>

        <div class="grid grid-flow-col-dense place-items-center place-content-center gap-4">
          {#each selectedPeople as person (person.id)}
            <div animate:flip={{ duration: 250, easing: quintOut }}>
              <FaceThumbnail border circle {person} selectable thumbnailSize={120} on:click={() => onSelect(person)} />
            </div>
          {/each}

          {#if hasSelection}
            <span><CallMerge size={48} class="rotate-90 dark:text-white" /> </span>
          {/if}
          <FaceThumbnail {person} border circle selectable={false} thumbnailSize={180} />
        </div>
      </div>
      <div
        class="p-10 overflow-y-auto rounded-3xl bg-gray-200 dark:bg-immich-dark-gray"
        style:max-height={screenHeight - 200 - 200 + 'px'}
      >
        <div class="grid grid-col-2 md:grid-cols-3 lg:grid-cols-6 xl:grid-cols-8 2xl:grid-cols-10 gap-8">
          {#each unselectedPeople as person (person.id)}
            <FaceThumbnail {person} on:click={() => onSelect(person)} circle border selectable />
          {/each}
        </div>
      </div>
    </section>

    {#if isShowConfirmation}
      <ConfirmDialogue
        title="Merge faces"
        confirmText="Merge"
        on:confirm={handleMerge}
        on:cancel={() => (isShowConfirmation = false)}
      >
        <svelte:fragment slot="prompt">
          <p>Are you sure you want merge these faces? <br />This action is <strong>irreversible</strong>.</p>
        </svelte:fragment>
      </ConfirmDialogue>
    {/if}
  </section>
</section>