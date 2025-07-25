<script setup lang="ts">
import { inject, onMounted } from 'vue';
import type { VocabAndTranslationRepoContract } from '@/entities/vocab/VocabAndTranslationRepoContract';
import type { ImmersionContentRepoContract } from '@/entities/immersion-content/ImmersionContentRepoContract';
import { useCachedQueue } from './useCachedQueue';
import PracticeVocabWidget from '@/features/practice-vocab/PracticeVocabWidget.vue';
import MetaTaskRenderer from './MetaTaskRenderer.vue';

// Inject repositories
const vocabRepo = inject<VocabAndTranslationRepoContract>('vocabRepo');
const immersionRepo = inject<ImmersionContentRepoContract>('immersionRepo');

if (!vocabRepo || !immersionRepo) {
  throw new Error('Repositories not available');
}

const {
  state,
  initializeQueue,
  completeCurrentVocab,
  completeCurrentTask,
  stateMachineDebug
} = useCachedQueue(vocabRepo, immersionRepo);

onMounted(async () => {
  await initializeQueue();
});

const handleVocabFinished = async () => {
  console.log('PageQueue: Vocab finished event received');
  await completeCurrentVocab();
  console.log('PageQueue: Completed vocab, new state:', state.value.status);
};

const handleTaskFinished = async () => {
  console.log('PageQueue: Task finished event received');
  await completeCurrentTask();
  console.log('PageQueue: Completed task, new state:', state.value.status);
};
</script>

<template>
  <div class="min-h-screen bg-base-200 p-4">
    <div class="max-w-4xl mx-auto">
      <!-- Loading State -->
      <div v-if="state.status === 'initializing' || state.status === 'loading'" class="flex justify-center items-center min-h-96">
        <div class="text-center">
          <span class="loading loading-spinner loading-lg"></span>
          <p class="mt-4 text-lg">
            {{ state.status === 'loading' && state.message ? state.message : 'Loading your learning queue...' }}
          </p>
        </div>
      </div>

      <!-- Error State -->
      <div v-else-if="state.status === 'error'" class="alert alert-error">
        <span>{{ state.message }}</span>
        <button class="btn btn-sm" @click="stateMachineDebug.retry">
          Retry
        </button>
      </div>

      <!-- No Content Available -->
      <div v-else-if="state.status === 'empty'" class="hero min-h-96">
        <div class="hero-content text-center">
          <div class="max-w-md">
            <h1 class="text-5xl font-bold">All Done!</h1>
            <p class="py-6">{{ state.message }}</p>
            <button class="btn btn-primary" @click="initializeQueue">
              Check Again
            </button>
          </div>
        </div>
      </div>

      <!-- Vocab Exercise -->
      <div v-else-if="state.status === 'vocab'">
        <div class="mb-4 text-center">
          <div class="breadcrumbs">
            <ul>
              <li>Queue</li>
              <li>Vocabulary Practice</li>
            </ul>
          </div>
          <div class="text-sm text-base-content/70">
            {{ state.batchIndex }} of {{ state.batchSize }} in current batch
          </div>
        </div>
        
        <PracticeVocabWidget 
          :key="state.vocab.id"
          :vocab="state.vocab"
          @finished="handleVocabFinished"
        />
      </div>

      <!-- Task -->
      <div v-else-if="state.status === 'task'">
        <div class="mb-4 text-center">
          <div class="breadcrumbs">
            <ul>
              <li>Queue</li>
              <li>Task</li>
            </ul>
          </div>
        </div>
        
        <MetaTaskRenderer 
          :task="state.task"
          @finished="handleTaskFinished"
        />
      </div>

      <!-- Fallback (should never happen with state machine) -->
      <div v-else class="alert alert-warning">
        <span>Unknown queue state. Please refresh.</span>
        <button class="btn btn-sm" @click="initializeQueue">
          Refresh
        </button>
      </div>

      <!-- Debug Info (in development) -->
      <!-- Uncomment during development if needed -->
      <!-- 
      <div class="mt-8 p-4 bg-base-300 rounded-lg text-xs">
        <details>
          <summary class="cursor-pointer">Debug Queue State</summary>
          <pre class="mt-2">{{ JSON.stringify({ state, preloaderStatus }, null, 2) }}</pre>
        </details>
      </div>
      -->
    </div>
  </div>
</template>