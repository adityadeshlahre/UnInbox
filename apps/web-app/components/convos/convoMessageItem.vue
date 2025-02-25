<script setup lang="ts">
  import { useTimeAgo } from '@vueuse/core';
  import { generateHTML } from '@tiptap/html';
  import { tipTapExtensions } from '../../shared/editorConfig';
  const { $trpc } = useNuxtApp();

  type ConvoEntryDataType = Awaited<
    ReturnType<typeof $trpc.convos.entries.getConvoEntries.query>
  >['entries'];

  type ConvoEntryItem = NonNullable<ConvoEntryDataType>[number];
  type Props = {
    entry: ConvoEntryItem;
  };

  const props = defineProps<Props>();
  const participantPublicId = inject('participantPublicId');
  const convoParticipants = inject('convoParticipants');

  const timeAgo = useTimeAgo(props.entry.createdAt || new Date());

  const convoEntryBody = computed(() => {
    if (props.entry.body) {
      return generateHTML(props.entry.body, tipTapExtensions);
    }
    return '';
  });

  const author = computed(() => {
    //@ts-expect-error vue3 provide/inject types are whack to set up: https://vuejs.org/guide/typescript/composition-api#typing-provide-inject
    return convoParticipants.value.find(
      //@ts-expect-error ts dosnt know the type of this reactive string
      (participant) =>
        participant.participantPublicId == props.entry.author.publicId
    );
  });
  const userIsAuthor = computed(() => {
    //@ts-expect-error ts dosnt know the type of this reactive string
    return props.entry.author.publicId == participantPublicId.value;
  });

  const tempColor = 'message';
  const typeClasses = computed(() => {
    switch (tempColor) {
      case 'message':
        return 'bg-white dark:bg-black';
      default:
        return 'bg-gray-100 dark:bg-gray-900';
    }
  });
  const containerClasses = computed(() => {
    return userIsAuthor.value ? 'items-end' : 'items-start';
  });

  const convoBubbleClasses = computed(() => {
    return `${typeClasses.value}`;
  });
</script>
<template>
  <div
    class="w-full flex flex-col gap-2"
    :class="containerClasses">
    <div class="w-full flex flex-row items-start gap-2">
      <div
        v-if="author"
        class="flex flex-row items-center gap-1">
        <UnUiAvatar
          :public-id="author.publicId"
          :avatar-id="author.avatarPublicId"
          :alt="author.name"
          :type="author.type"
          :color="author.color"
          size="lg"
          class="rounded-full" />
      </div>
      <div class="w-full flex flex-col gap-2 p-0">
        <div class="w-full flex flex-row items-end gap-2 pl-2">
          <span
            v-if="author"
            class="text-sm leading-none">
            {{ author.name }}
          </span>
          <UnUiTooltip :text="props.entry.createdAt.toLocaleString()">
            <span class="text-gray-500 text-xs leading-none">
              {{ timeAgo }}
            </span>
          </UnUiTooltip>
        </div>
        <div
          class="bg-gray-100 dark:bg-gray-900 w-full overflow-hidden rounded-bl-xl rounded-br-xl p-2"
          :class="convoBubbleClasses"
          v-html="convoEntryBody"></div>
      </div>
    </div>
  </div>
</template>
