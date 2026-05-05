<script setup lang="ts">
import { computed, ref, watch } from 'vue';
import { useAuthStore } from '@/stores/auth';

interface VForm {
  validate: () => Promise<{ valid: boolean }>;
  resetValidation: () => void;
}

const model = defineModel<boolean>({ required: true });
const emit = defineEmits<{ success: [] }>();

const auth = useAuthStore();

const username = computed(() => auth.user?.email ?? '');

const oldPassword = ref('');
const newPassword = ref('');
const showOldPassword = ref(false);
const showNewPassword = ref(false);
const formRef = ref<VForm | null>(null);
const isSubmitting = ref(false);
const errorMessage = ref<string | null>(null);

const oldPasswordRules = [
  (v: string): true | string => (!!v && v.length > 0) || 'Old password is required',
];

const newPasswordRules = [
  (v: string): true | string => (!!v && v.length > 0) || 'New password is required',
  (v: string): true | string => v.length >= 6 || 'New password must be at least 6 characters',
  (v: string): true | string =>
    v !== oldPassword.value || 'New password must be different from the old password',
];

const passwordStrength = computed(() => {
  const v = newPassword.value;
  if (!v) return { score: 0, label: '', color: 'grey' };

  let score = 0;
  if (v.length >= 6) score += 1;
  if (v.length >= 10) score += 1;
  if (/[A-Z]/.test(v) && /[a-z]/.test(v)) score += 1;
  if (/\d/.test(v)) score += 1;
  if (/[^A-Za-z0-9]/.test(v)) score += 1;

  if (score <= 1) return { score: 20, label: 'Weak', color: 'error' };
  if (score === 2) return { score: 45, label: 'Fair', color: 'warning' };
  if (score === 3) return { score: 70, label: 'Good', color: 'info' };
  return { score: 100, label: 'Strong', color: 'success' };
});

const resetForm = (): void => {
  oldPassword.value = '';
  newPassword.value = '';
  showOldPassword.value = false;
  showNewPassword.value = false;
  errorMessage.value = null;
  formRef.value?.resetValidation();
};

watch(model, (open) => {
  if (open) resetForm();
});

const onCancel = (): void => {
  if (isSubmitting.value) return;
  model.value = false;
};

const onConfirm = async (): Promise<void> => {
  if (!formRef.value) return;
  const { valid } = await formRef.value.validate();
  if (!valid) return;

  isSubmitting.value = true;
  errorMessage.value = null;
  try {
    const result = await auth.changePassword(oldPassword.value, newPassword.value);
    if (result.success) {
      model.value = false;
      emit('success');
    } else {
      errorMessage.value = result.message ?? 'Failed to change password. Please try again.';
    }
  } finally {
    isSubmitting.value = false;
  }
};
</script>

<template>
  <v-dialog v-model="model" max-width="420" :persistent="isSubmitting">
    <v-card rounded="lg" class="overflow-hidden">
      <div class="d-flex align-center px-5 pt-4 pb-2">
        <v-avatar size="36" color="primary" variant="tonal" class="mr-3">
          <v-icon size="20">mdi-lock-reset</v-icon>
        </v-avatar>
        <div class="flex-grow-1">
          <div class="text-subtitle-1 font-weight-bold">Change password</div>
          <div class="text-caption text-medium-emphasis">
            Enter your current and new password.
          </div>
        </div>
      </div>

      <v-card-text class="pt-3 pb-2 px-5">
        <v-alert
          v-if="errorMessage"
          type="error"
          variant="tonal"
          density="compact"
          class="mb-4"
          closable
          @click:close="errorMessage = null"
        >
          {{ errorMessage }}
        </v-alert>

        <v-form ref="formRef" :disabled="isSubmitting" @submit.prevent="onConfirm">
          <input
            type="text"
            name="username"
            autocomplete="username"
            :value="username"
            readonly
            aria-hidden="true"
            tabindex="-1"
            class="visually-hidden-username"
          />

          <v-text-field
            v-model="oldPassword"
            label="Old password"
            :type="showOldPassword ? 'text' : 'password'"
            autocomplete="current-password"
            :rules="oldPasswordRules"
            prepend-inner-icon="mdi-lock-outline"
            :append-inner-icon="showOldPassword ? 'mdi-eye-off' : 'mdi-eye'"
            required
            @click:append-inner="showOldPassword = !showOldPassword"
          />

          <v-text-field
            v-model="newPassword"
            label="New password"
            :type="showNewPassword ? 'text' : 'password'"
            autocomplete="new-password"
            :rules="newPasswordRules"
            prepend-inner-icon="mdi-lock-plus-outline"
            :append-inner-icon="showNewPassword ? 'mdi-eye-off' : 'mdi-eye'"
            required
            @click:append-inner="showNewPassword = !showNewPassword"
            @keyup.enter="onConfirm"
          />

          <div v-if="newPassword" class="strength-meter mt-1 mb-2">
            <div class="d-flex justify-space-between align-center mb-1">
              <span class="text-caption text-medium-emphasis">Password strength</span>
              <span :class="`text-caption text-${passwordStrength.color} font-weight-medium`">
                {{ passwordStrength.label }}
              </span>
            </div>
            <v-progress-linear
              :model-value="passwordStrength.score"
              :color="passwordStrength.color"
              height="6"
              rounded
            />
          </div>
        </v-form>
      </v-card-text>

      <v-card-actions class="px-5 pb-4 pt-2">
        <v-btn
          variant="text"
          size="large"
          class="flex-1-1-0"
          :disabled="isSubmitting"
          @click="onCancel"
        >
          Cancel
        </v-btn>
        <v-btn
          color="primary"
          variant="flat"
          size="large"
          class="flex-1-1-0"
          :loading="isSubmitting"
          @click="onConfirm"
        >
          Confirm
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<style scoped>
.strength-meter {
  user-select: none;
}

.visually-hidden-username {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
</style>
