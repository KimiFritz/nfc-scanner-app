<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
  <ion-title>NFC Scanner-App</ion-title>
  <!-- NFC support icon in the toolbar -->
  <ion-button v-if="isSupported !== null" id="support-icon" slot="end" fill="clear" aria-label="NFC support">
          <ion-icon :name="getSupportIconName()" :class="['support-icon', supportClass]"></ion-icon>
        </ion-button>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">NFC Scanner-App</ion-title>
        </ion-toolbar>
      </ion-header>

      <!-- Intro section -->
      <div class="intro">
        <ion-icon name="radio-outline" class="intro-icon" aria-hidden="true"></ion-icon>
        <div class="intro-text">Read & write NFC tags</div>
      </div>

      <!-- Popover for NFC support information -->
      <ion-popover v-if="isSupported !== null" trigger="support-icon" trigger-action="click hover">
        <div class="popover-content" :class="supportClass">
          {{ supportMessage }}
        </div>
      </ion-popover>

      <!-- Actions -->
      <div class="actions">
        <ion-button expand="block" @click="readNfc" :disabled="!isSupported || isBusy">
          <ion-icon name="scan-outline" slot="start"></ion-icon>
          Read
        </ion-button>
        <ion-button expand="block" @click="writeNfc" :disabled="!isSupported || isBusy">
          <ion-icon name="create-outline" slot="start"></ion-icon>
          Write
        </ion-button>
        <ion-button v-if="isDev" expand="block" color="medium" @click="openNfcDetailDemo">
          Open detail demo
        </ion-button>
      </div>
    
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton, IonIcon, IonPopover } from '@ionic/vue';
import { Nfc, NfcUtils } from '@capawesome-team/capacitor-nfc';
import { alertController, toastController } from '@ionic/vue';
import { Capacitor } from '@capacitor/core';
import { onMounted, ref, computed } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();

// App support status + busy flag
const isSupported = ref<boolean | null>(null); // null = checking
const isBusy = ref(false);

// Icon to visualize NFC support status
const getSupportIconName = () => (isSupported.value ? 'checkmark-circle-outline' : 'close-circle-outline');

const supportClass = computed(() => (isSupported.value ? 'supported' : 'unsupported'));

// Status message used inside the popover
const supportMessage = computed(() => (isSupported.value ? 'This device supports NFC.' : 'This device does not support NFC.'));

// On app start: check if device supports NFC (capability only)
onMounted(async () => {
  await checkDeviceSupport();
});

/**
 * Check basic NFC capability of the device (no permission/enabled checks here).
 */
const checkDeviceSupport = async () => {
  try {
    const { isSupported: supported } = await Nfc.isSupported();
  isSupported.value = !!supported;
  } catch (error) {
   console.error('Error checking NFC support:', error);
  isSupported.value = false;
  }
};

/**
 * Generic alert helper that presents an alert and returns it.
 */
const presentAlert = async (opts: Parameters<typeof alertController.create>[0]) => {
  const alert = await alertController.create(opts);
  await alert.present();
  return alert;
};

/** Present a simple error alert. */
const showErrorDialog = async (header: string, message: string) => {
  const alert = await alertController.create({
    header,
    message,
    buttons: ['OK']
  });
  await alert.present();
};

/**
 * Explain why NFC permission is needed and allow opening settings (Android).
 */
const showPermissionDialog = async () => {
  const alert = await alertController.create({
    header: 'NFC permission required',
    message: 'This app needs NFC permission to read and write tags.',
    buttons: [
      { text: 'Cancel', role: 'cancel' },
      { 
        text: 'Open settings', 
        handler: async () => {
          if (Capacitor.getPlatform() === 'android') {
            await Nfc.openSettings();
          }
        }
      }
    ]
  });
  await alert.present();
};

/**
 * Ask the user to enable NFC in Android settings when it is disabled.
 */
const showNfcDeactivationDialog = async () => {
  const alert = await alertController.create({
    header: 'Enable NFC',
    message: 'NFC is disabled on this device. Please enable it in Settings to use this app.',
    buttons: [
      { 
        text: 'Cancel', 
        role: 'cancel',
  handler: () => {}
      },
      { 
        text: 'Open settings', 
        handler: async () => {
          await Nfc.openSettings();
          // Optional: re-check base support (isEnabled is checked on action)
          setTimeout(() => checkDeviceSupport(), 1000);
        }
      }
    ]
  });
  await alert.present();
};


/**
 * Ensure NFC is enabled (Android) and the permission is granted.
 * Only called on user actions (read/write), not on app start.
 */
const ensureEnabledAndPermission = async (): Promise<void> => {
  // Use cached status; if unknown, perform a single check
  if (isSupported.value === null) {
    await checkDeviceSupport();
  }
  if (!isSupported.value) {
   await showErrorDialog('Not supported', 'This device does not support NFC.');
    throw new Error('unsupported');
  }

  if (Capacitor.getPlatform() === 'android') {
    const { isEnabled } = await Nfc.isEnabled();
    if (!isEnabled) {
      await showNfcDeactivationDialog();
      throw new Error('disabled');
    }
  }

  // Funktion eigentlich nur für Web notwendig (siehe Dokumentation)
  const { nfc } = await Nfc.checkPermissions();
  if (nfc !== 'granted') {
    const { nfc: requestedPermission } = await Nfc.requestPermissions();
    if (requestedPermission !== 'granted') {
  await showPermissionDialog();
      throw new Error('permission-denied');
    }
  }
};

/**
 * Start an NFC scan session and wait for a tag.
 * Provides a cancel alert and ensures proper cleanup of listeners/session.
 */
const waitForTag = async (header = 'NFC', message = 'Hold an NFC tag near the device') => {
  const blocker = await presentAlert({
    header,
    message,
    backdropDismiss: false,
  buttons: [{ text: 'Cancel', role: 'cancel' }]
  });

  const subs: Array<{ remove: () => Promise<void> }> = [];
  try {
    await Nfc.removeAllListeners();

    const tagP = new Promise<any>((resolve, reject) => {
      Nfc.addListener('nfcTagScanned', (e) => resolve(e.nfcTag)).then(s => subs.push(s));
    Nfc.addListener('scanSessionError', (e) => reject(new Error(e?.message || 'Scan error'))).then(s => subs.push(s));
    });

    const cancelP = new Promise<never>((_, reject) => {
      blocker.onDidDismiss().then((detail) => {
        if (detail.role === 'cancel') {
      // Stop session and reject
          Nfc.stopScanSession().finally(() => reject(new Error('cancelled')));
        }
      });
    });

    await Nfc.startScanSession();
    return await Promise.race([tagP, cancelP]);
  } finally {
    try { await Nfc.stopScanSession(); } catch {}
    await Promise.allSettled(subs.map(s => s.remove()));
    await blocker.dismiss();
  }
};

/**
 * Normalize the raw tag data to a compact object used for navigation.
 */
const utils = new NfcUtils();
const normalizeTag = (tag: any) => {
  const id = tag?.id ? utils.convertBytesToHex({ bytes: tag.id }).hex : 'Unknown';
  let payload = '';
  const rec = tag?.message?.records?.[0];
  if (rec?.payload) {
    try {
      const { text } = utils.getTextFromNdefTextRecord({ record: rec });
      payload = text || utils.convertBytesToString({ bytes: rec.payload }).text;
    } catch {
      payload = utils.convertBytesToString({ bytes: rec.payload }).text;
    }
  }
  return {
    id,
    payload,
    isWritable: !!tag?.isWritable,
    idBytes: tag?.id || [],
    payloadBytes: rec?.payload || [],
    techTypes: tag?.techTypes || [],
    maxSize: tag?.maxSize || 0,
    type: tag?.type || 'Unknown'
  };
};


/** Read an NFC tag and navigate to the detail page. */
const readNfc = async () => {
  if (!isSupported.value || isBusy.value) return;
  isBusy.value = true;
  try {
    await ensureEnabledAndPermission();
    const tag = await waitForTag('Scan NFC tag', 'Hold the NFC tag near the device');
    const tagData = normalizeTag(tag);
    router.push({ name: 'nfc-detail-json', query: { tag: JSON.stringify(tagData) } });
  } catch (error) {
    const t = await toastController.create({
      message: 'Error reading the NFC tag',
      duration: 2000,
      position: 'bottom',
      color: 'danger'
    });
    await t.present();
  } finally {
    isBusy.value = false;
  }
};

/** Prompt for text and write it to an NFC tag. */
const writeNfc = async () => {
  if (!isSupported.value || isBusy.value) return;
  isBusy.value = true;
  try {
    await ensureEnabledAndPermission();

    const ask = await presentAlert({
      header: 'Write NFC tag',
      message: 'Please enter the text to write to the NFC tag:',
      inputs: [{ name: 'nfcText', type: 'text', placeholder: 'Text for NFC tag' }],
      buttons: [{ text: 'Cancel', role: 'cancel' }, { text: 'OK', role: 'confirm' }]
    });
    const { data, role } = await ask.onDidDismiss();
    const text: string | undefined = data?.values?.nfcText?.trim();
    if (role !== 'confirm' || !text) return;

    const { record } = utils.createNdefTextRecord({ text });
    await waitForTag('Waiting for NFC tag…', 'Hold the NFC tag near the device to write the text.');
    await Nfc.write({ message: { records: [record] } });

    await presentAlert({ header: 'Success', message: 'Text has been written to the NFC tag.', buttons: ['OK'] });
  } catch (error) {
    const t = await toastController.create({
      message: 'Error writing to the NFC tag',
      duration: 2000,
      position: 'bottom',
      color: 'danger'
    });
    await t.present();
  } finally {
    isBusy.value = false;
  }
};

const isDev = import.meta.env.DEV;

/** Navigate to the detail page with mock data (dev only). */
const openNfcDetailDemo = () => {
  const id = '01020304';
  const payload = 'Hello NFC';
  const isWritable = true;
  const idBytes = [1, 2, 3, 4];
  const payloadBytes = [72, 101, 108, 108, 111, 32, 78, 70, 67]; // "Hello NFC"
  const techTypes: string[] = [];
  const maxSize = 0;
  const type = 'Unknown';

  router.push({ name: 'nfc-detail-json', query: { tag: JSON.stringify({
    id,
    payload,
    isWritable,
    idBytes,
    payloadBytes,
    techTypes,
    maxSize,
    type
  }) } });
};
</script>


<style scoped>
.support-icon { font-size: 25px; }
.support-icon.supported { color: #2e7d32; }
.support-icon.unsupported { color: #d32f2f; }

.popover-content {
  padding: 10px 12px;
  font-size: 14px;
  min-width: 220px;
}

.popover-content.supported { color: #2e7d32; }
.popover-content.unsupported { color: #d32f2f; }

.intro {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 24px 0 8px;
}

.intro-icon {
  font-size: 72px;
  color: var(--ion-color-primary);
  opacity: 0.85;
}

.intro-text {
  margin-top: 6px;
  color: var(--ion-color-dark);
  font-size: 16px;
}

.actions {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 12px;
}
</style>
