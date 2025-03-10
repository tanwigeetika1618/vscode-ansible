<script setup lang="ts">
import { ref, watch } from 'vue';
import type { Ref } from 'vue'
import { vscodeApi } from './utils';
import { allComponents, provideVSCodeDesignSystem } from '@vscode/webview-ui-toolkit';

import { RoleGenerationResponseParams } from "../../../../src/interfaces/lightspeed";

import SavedFiles from "./components/SavedFiles.vue";
import StatusBox from './components/StatusBox.vue';
import OutlineReview from './components/OutlineReview.vue';
import GeneratedFileEntry from './components/GeneratedFileEntry.vue';
import CollectionSelector from "./components/CollectionSelector.vue";
import ErrorBox from './components/ErrorBox.vue';
import PromptExampleBox from './components/PromptExampleBox.vue';
import PromptField from './components/PromptField.vue';

provideVSCodeDesignSystem().register(allComponents);

const page = ref(1);
const prompt = ref('');
const collectionName = ref('');
const roleName = ref('');
const response: Ref<RoleGenerationResponseParams | undefined> = ref();
const outline = ref('');
const errorMessages: Ref<string[]> = ref([])
const loadingNewResponse = ref(false);
const filesWereSaved = ref(false);

async function nextPage() {
  if (response.value !== undefined) {
    page.value++;
    return;
  }
  loadingNewResponse.value = true;
  await vscodeApi.post('generateRole', { text: prompt.value, outline: outline.value });

}

vscodeApi.on('generateRole', (data: any) => {
  response.value = undefined; // To disable the watchers
  roleName.value = data["role"];
  outline.value = data["outline"] || outline.value;
  if (Array.isArray(data["warnings"])) {
    errorMessages.value.push(...data["warnings"]);
  }
  response.value = data as RoleGenerationResponseParams;
  loadingNewResponse.value = false;
  page.value++;
});



// Reset some stats before the page transition
watch(page, (newPage) => {
  errorMessages.value = [];
  filesWereSaved.value = false;
  if (newPage === 1) {
    roleName.value = "";
  }
})

watch(prompt, (newPrompt) => {
  if (response.value !== undefined) {
    response.value = undefined;
    outline.value = "";
  }
})

watch(roleName, (newRoleName) => {
  if (response.value !== undefined && response.value["role"] !== newRoleName) {
    response.value = undefined;
  }
})

watch(outline, (newOutline) => {
  if (response.value !== undefined && response.value["outline"] !== newOutline) {
    response.value = undefined;
  }
})


</script>

<template>
  <h2 id="main-header">Create a role with Ansible Lightspeed</h2>
  <div class="pageNumber" id="page-number">{{ page }} of 3</div>

  <ErrorBox v-model:error-messages="errorMessages" />

  <div id="roleInfo">
    <a href="https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html">Learn more about
      roles🔗</a>
  </div>

  <ProgressSpinner v-if="loadingNewResponse" />

  <div v-else-if="page === 1">
    <CollectionSelector v-model:collection-name="collectionName" v-model:error-messages="errorMessages" />
    <PromptField v-model:prompt="prompt" />

    <div>
      <vscode-button @click.once="nextPage" :disabled="prompt === '' || collectionName === ''">
        Analyze
      </vscode-button>
    </div>
    <PromptExampleBox />
  </div>

  <div v-else-if="page === 2">
    <StatusBox :prompt="prompt" :collectionName="collectionName" @restart-wizard="page = 1" />
    <div>
      Role name: <vscode-textfield v-model="roleName" />
    </div>

    <OutlineReview :outline
      @outline-update="(newOutline: string) => { console.log(`new outline: ${newOutline}`); outline = newOutline; }" />

    <div>
      <vscode-button @click.once="nextPage">
        Continue
      </vscode-button>
      <vscode-button secondary @click="page--">
        Back
      </vscode-button>
    </div>
  </div>

  <div v-else-if="page === 3">
    <GeneratedFileEntry v-for="file in response?.files" :file />
    <Suspense>
      <SavedFiles v-if="filesWereSaved && response" :files="response.files" :role-name="roleName"
        :collection-name="collectionName" />
      <template #fallback>
        <ProgressSpinner />
      </template>
    </Suspense>
    <div>
      <vscode-button @click="filesWereSaved = true">
        Save files
      </vscode-button>
      <vscode-button secondary @click="page--">
        Back
      </vscode-button>
    </div>
  </div>
</template>

<style scoped>
#role-prompt {
  width: 100ch;
}
</style>
