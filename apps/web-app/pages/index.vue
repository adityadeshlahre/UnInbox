<script setup lang="ts">
  import { startAuthentication } from '@simplewebauthn/browser';
  import { z } from 'zod';
  const { $trpc, $i18n } = useNuxtApp();
  definePageMeta({ guest: true });
  const turnstileToken = ref();
  const errorMessage = ref(false);
  const passkeyLocation = ref('');
  const immediatePasskeyPrompt = ref(false);

  const showPasswordFields = ref(false);

  //Form Fields
  const usernameValid = ref<boolean | 'remote' | null>(null);
  const usernameValue = ref('');
  const usernameValidationMessage = ref('');
  const passwordInput = ref('');
  const passwordValid = ref<boolean | null>(null);
  const passwordValidationMessage = ref('');

  const formValid = computed(() => {
    return usernameValid.value === true && passwordValid.value === true;
  });

  const disablePasswordButton = computed(() => {
    if (!showPasswordFields.value) {
      return false;
    }
    return !formValid.value;
  });

  const turnstileEnabled = useRuntimeConfig().public.turnstileEnabled;
  if (!turnstileEnabled) {
    turnstileToken.value = '';
  }

  if (process.client) {
    passkeyLocation.value = localStorage.getItem('passkeyLocation') || '';
    immediatePasskeyPrompt.value = JSON.parse(
      localStorage.getItem('immediatePasskeyPrompt') || 'false'
    );
  }

  watch(immediatePasskeyPrompt, async () => {
    const newValue = JSON.stringify(immediatePasskeyPrompt.value);
    localStorage.setItem('immediatePasskeyPrompt', newValue);
  });
  let timeoutId: NodeJS.Timeout | null = null;

  async function passwordButton() {
    if (!showPasswordFields.value) {
      showPasswordFields.value = true;
      return;
    }
    await doPasswordLogin();
  }

  async function doPasswordLogin() {
    const passwordVerification =
      await $trpc.auth.password.passwordSignIn.mutate({
        turnstileToken: turnstileToken.value,
        username: usernameValue.value,
        password: passwordInput.value
      });
    if (passwordVerification.success) {
      navigateTo('/redirect');
    }
  }

  async function doPasskeyLogin() {
    if (turnstileEnabled && !turnstileToken.value) {
      errorMessage.value = true;

      await new Promise((resolve) => {
        const unwatch = watch(turnstileToken, (newValue) => {
          if (newValue !== null) {
            resolve(newValue);
            unwatch();
            errorMessage.value = false;
          }
        });
      });
    }
    const toast = useToast();
    if (timeoutId !== null) {
      clearTimeout(timeoutId);
      timeoutId = null;
    }

    const passkeyOptions =
      await $trpc.auth.passkey.generatePasskeyChallenge.query({
        turnstileToken: turnstileToken.value
      });

    if (turnstileEnabled) {
      turnstileToken.value.reset();
    }

    if (!passkeyOptions.options) {
      toast.add({
        title: 'Server error',
        description:
          'We couldnt generate a secure login for you, please check your internet connection.',
        color: 'red',
        timeout: 5000,
        icon: 'i-ph-warning-circle'
      });
      return;
    }
    try {
      const passkeyData = await startAuthentication(passkeyOptions.options);

      if (!passkeyData) {
        throw new Error('No passkey data returned');
      }

      const verifyPasskey = await $trpc.auth.passkey.verifyPasskey.mutate({
        turnstileToken: turnstileToken.value,
        verificationResponseRaw: passkeyData
      });
      if (verifyPasskey.success) {
        navigateTo('/redirect');
      }
    } catch (e) {
      toast.add({
        title: 'Passkey error',
        description:
          'Something went wrong when getting your passkey, please try again.',
        color: 'red',
        timeout: 5000,
        icon: 'i-ph-warning-circle'
      });
    }
  }

  const pageReady: Ref<boolean> = ref(false);
  onNuxtReady(() => {
    pageReady.value = true;
  });
</script>

<template>
  <div
    class="h-screen w-screen flex flex-col items-center justify-between p-4 pb-14">
    <div
      class="max-w-48 w-full flex grow flex-col items-center justify-center gap-8 md:max-w-72">
      <h1 class="mb-4 text-center text-2xl font-display">
        Login to your <br /><span class="text-5xl">UnInbox</span>
      </h1>

      <div class="w-full flex flex-col gap-4">
        <UnUiButton
          label="Login with my passkey"
          icon="i-ph-key-duotone"
          block
          color="primary"
          size="lg"
          @click="doPasskeyLogin()" />
        <UnUiDivider label="or" />
        <div class="w-full flex flex-col gap-2 -mt-2">
          <div
            class="transition-[max-height] w-full flex flex-col gap-2 overflow-hidden duration-800 ease-in-out"
            :class="showPasswordFields ? 'max-h-96' : 'max-h-0'">
            <UnUiInput
              v-model:value="usernameValue"
              v-model:valid="usernameValid"
              v-model:validationMessage="usernameValidationMessage"
              width="full"
              icon="i-ph-at"
              label="Username"
              helper="Can only contain letters and numbers."
              placeholder=""
              :schema="
                z
                  .string()
                  .min(2, { message: 'Must be at least 5 characters long' })
                  .max(32, {
                    message: 'Too Long, Aint nobody typing that 😂'
                  })
                  .regex(/^[a-zA-Z0-9]*$/, {
                    message: 'Only letters and numbers'
                  })
              " />
            <UnUiInput
              v-model:value="passwordInput"
              v-model:valid="passwordValid"
              v-model:validationMessage="passwordValidationMessage"
              width="full"
              icon="i-ph-password"
              label="Password"
              password
              placeholder=""
              :schema="
                z
                  .string()
                  .min(8, { message: 'Minimum 8 characters' })
                  .max(64)
                  .regex(
                    /^(?=.*\d)(?=.*[a-z])(?=.*[A-Z])(?=.*\W)(?!.*\s).{8,}$/,
                    {
                      message:
                        'At least one digit, one lowercase letter, one uppercase letter, one special character, no whitespace allowed, minimum eight characters in length'
                    }
                  )
              " />
          </div>
          <UnUiButton
            label="Login with my password"
            icon="i-ph-password"
            block
            variant="outline"
            size="lg"
            :disabled="disablePasswordButton"
            @click="passwordButton()" />
        </div>
      </div>
      <UnUiButton
        label="Not a member yet? Join instead"
        variant="soft"
        block
        size="lg"
        @click="navigateTo('/join')" />
      <UnUiButton
        label="I lost my passkey"
        variant="ghost"
        block
        @click="navigateTo('/login/findmypasskey')" />
      <div class="h-0 max-h-0 max-w-full w-full">
        <UnUiAlert
          v-show="errorMessage"
          icon="i-ph-warning-circle"
          title="Waiting for human verification!"
          description="This is an automated process and should complete within a few seconds. If it doesn't, please refresh the page."
          color="orange"
          variant="solid" />
      </div>
      <NuxtTurnstile
        v-if="pageReady && turnstileEnabled"
        v-model="turnstileToken"
        class="fixed bottom-5 mb-[-30px] scale-50 hover:(mb-0 scale-100)" />
    </div>
  </div>
</template>
