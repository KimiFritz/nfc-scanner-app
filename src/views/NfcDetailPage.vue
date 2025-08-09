<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-buttons slot="start">
          <ion-back-button default-href="/"></ion-back-button>
        </ion-buttons>
  <ion-title>NFC Tag Details</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">NFC Tag Details</ion-title>
        </ion-toolbar>
      </ion-header>

      <!-- Intro section aligned with HomePage -->
      <div class="intro">
        <ion-icon name="radio-outline" class="intro-icon" aria-hidden="true"></ion-icon>
        <div class="intro-text">Details of the scanned NFC tag</div>
      </div>

      <div class="detail-container">
    <!-- Tag-ID -->
        <ion-card>
          <ion-card-header>
      <ion-card-title>Tag-ID</ion-card-title>
          </ion-card-header>
          <ion-card-content>
            <p class="detail-value">{{ tagData.id }}</p>
            <p class="detail-bytes" v-if="tagData.idHex">Hex: {{ tagData.idHex }}</p>
          </ion-card-content>
        </ion-card>

        <!-- Payload -->
        <ion-card>
          <ion-card-header>
            <ion-card-title>Payload</ion-card-title>
          </ion-card-header>
          <ion-card-content>
            <p class="detail-value" v-if="tagData.payload">{{ tagData.payload }}</p>
      <p class="detail-empty" v-else>No payload available</p>
            <p class="detail-bytes" v-if="tagData.payloadHex">Hex: {{ tagData.payloadHex }}</p>
            
            <div class="action-buttons" v-if="tagData.payload">
              <ion-button fill="outline" size="small" @click="copyToClipboard">
                <ion-icon name="clipboard-outline" slot="start"></ion-icon>
        Copy
              </ion-button>
              <ion-button fill="outline" size="small" @click="sharePayload">
                <ion-icon name="share-outline" slot="start"></ion-icon>
        Share
              </ion-button>
            </div>
          </ion-card-content>
        </ion-card>

    <!-- Writable -->
        <ion-card>
          <ion-card-header>
      <ion-card-title>Writable</ion-card-title>
          </ion-card-header>
          <ion-card-content>
            <div class="writable-icon">
                <ion-icon v-if="tagData.isWritable" name="checkmark-circle-outline" class="status-icon" color="success"></ion-icon>
                <ion-icon v-else name="close-circle-outline" class="status-icon" color="danger"></ion-icon>
              <span class="writable-text">{{ tagData.isWritable ? 'Yes' : 'No' }}</span>
            </div>
          </ion-card-content>
        </ion-card>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButtons, IonBackButton, IonCard, IonCardHeader, IonCardTitle, IonCardContent, IonButton, IonIcon, toastController } from '@ionic/vue';
import { Clipboard } from '@capacitor/clipboard';
import { Share } from '@capacitor/share';
import { NfcUtils } from '@capawesome-team/capacitor-nfc';
import { useRoute } from 'vue-router';
import { onMounted, ref } from 'vue';

const route = useRoute();

// Tag data
interface TagData {
  id: string;
  payload: string;
  isWritable: boolean;
  idBytes: number[];
  payloadBytes: number[];
  idHex: string;
  payloadHex: string;
}

const tagData = ref<TagData>({
  id: '',
  payload: '',
  isWritable: false,
  idBytes: [],
  payloadBytes: [],
  idHex: '',
  payloadHex: ''
});

// Parse route parameters/query
onMounted(() => {
  try {
    const utils = new NfcUtils();
    // 1) Preferred: compact JSON via query ?tag=...
    const tagJson = (route.query.tag as string) || '';
    if (tagJson) {
      const parsed = JSON.parse(tagJson);
      tagData.value.id = parsed.id ?? 'Unknown';
      tagData.value.payload = parsed.payload ? String(parsed.payload) : '';
      tagData.value.isWritable = !!parsed.isWritable;
      tagData.value.idBytes = Array.isArray(parsed.idBytes) ? parsed.idBytes : [];
      tagData.value.payloadBytes = Array.isArray(parsed.payloadBytes) ? parsed.payloadBytes : [];
    } else {
      // 2) Fallback: legacy params
      tagData.value.id = (route.params.id as string) || 'Unknown';
      tagData.value.payload = decodeURIComponent((route.params.payload as string) || '');
      tagData.value.isWritable = route.params.isWritable === 'true';
      tagData.value.idBytes = JSON.parse((route.params.idBytes as string) || '[]');
      tagData.value.payloadBytes = JSON.parse((route.params.payloadBytes as string) || '[]');
    }

    if (tagData.value.idBytes.length > 0) {
      tagData.value.idHex = utils.convertBytesToHex({ bytes: tagData.value.idBytes, separator: ' ' }).hex;
    }
    if (tagData.value.payloadBytes.length > 0) {
      tagData.value.payloadHex = utils.convertBytesToHex({ bytes: tagData.value.payloadBytes, separator: ' ' }).hex;
    }
  } catch (error) {
    console.error('Fehler beim Verarbeiten der Tag-Daten:', error);
  }
});

/** Copy payload to clipboard and show a toast. */
const copyToClipboard = async () => {
  try {
    await Clipboard.write({ string: tagData.value.payload });
    const toast = await toastController.create({
      message: 'Payload copied to clipboard',
      duration: 2000,
      position: 'bottom',
      color: 'success'
    });
    await toast.present();
  } catch (error) {
    console.error('Error copying to clipboard:', error);
    const toast = await toastController.create({
      message: 'Failed to copy to clipboard',
      duration: 2000,
      position: 'bottom',
      color: 'danger'
    });
    await toast.present();
  }
};

/** Share the payload using the native share sheet (ignore user cancel). */
const sharePayload = async () => {
  try {
    await Share.share({
      title: 'NFC Tag Payload',
      text: tagData.value.payload,
      dialogTitle: 'Share NFC tag payload'
    });
  } catch (error) {
    // Ignore user cancel
    const name = ((error as any)?.name || '').toString().toLowerCase();
    const message = ((error as any)?.message || (error as any)?.toString?.() || '').toString().toLowerCase();
    const isCancel = name === 'aborterror' ||
      message.includes('abort') ||
      message.includes('aborted') ||
      message.includes('canceled') ||
      message.includes('cancelled') ||
      message.includes('user cancelled') ||
      message.includes('user canceled');
    if (isCancel) {
      return;
    }
    console.error('Error sharing payload:', error);
    const toast = await toastController.create({
      message: 'Failed to share payload',
      duration: 2000,
      position: 'bottom',
      color: 'danger'
    });
    await toast.present();
  }
};
</script>

<style scoped>
.intro {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px 0 8px;
}

.intro-icon {
  font-size: 76px;
  color: var(--ion-color-primary);
  opacity: 0.85;
}

.intro-text {
  margin-top: 6px;
  color: var(--ion-color-dark);
  font-size: 16px;
}

.detail-container {
  padding: 16px;
}

.detail-value {
  font-family: 'Courier New', monospace;
  font-size: 16px;
  font-weight: bold;
  color: var(--ion-color-primary);
  word-break: break-all;
  margin: 8px 0;
}

.detail-empty {
  font-style: italic;
  color: var(--ion-color-medium);
  margin: 8px 0;
}

.detail-bytes {
  font-family: 'Courier New', monospace;
  font-size: 14px;
  color: var(--ion-color-dark);
  word-break: break-all;
  margin: 4px 0;
}

.action-buttons {
  display: flex;
  gap: 8px;
  margin-top: 16px;
  flex-wrap: wrap;
}

ion-card { margin: 8px 0; }
ion-item { --padding-start: 0; --padding-end: 0; }

.writable-icon {
  display: flex;
  justify-content: left;
  align-items: center;
  padding: 8px 0;
  gap: 8px;
}
.status-icon { font-size: 28px; }

.writable-text {
  font-size: 16px;
}

</style>
