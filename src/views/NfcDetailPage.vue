<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-buttons slot="start">
          <ion-back-button default-href="/"></ion-back-button>
        </ion-buttons>
        <ion-title>NFC-Tag Details</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">NFC-Tag Details</ion-title>
        </ion-toolbar>
      </ion-header>

      <div class="detail-container">
        <!-- Tag ID -->
        <ion-card>
          <ion-card-header>
            <ion-card-title>Tag ID</ion-card-title>
          </ion-card-header>
          <ion-card-content>
            <p class="detail-value">{{ tagData.id }}</p>
            <p class="detail-bytes">Bytes: {{ tagData.idHex }}</p>
          </ion-card-content>
        </ion-card>

        <!-- Payload -->
        <ion-card>
          <ion-card-header>
            <ion-card-title>Payload</ion-card-title>
          </ion-card-header>
          <ion-card-content>
            <p class="detail-value" v-if="tagData.payload">{{ tagData.payload }}</p>
            <p class="detail-empty" v-else>Kein Payload verfügbar</p>
            <p class="detail-bytes" v-if="tagData.payloadHex">Bytes: {{ tagData.payloadHex }}</p>
            
            <!-- Aktions-Buttons für Payload -->
            <div class="action-buttons" v-if="tagData.payload">
              <ion-button fill="outline" size="small" @click="copyToClipboard">
                <ion-icon slot="start" :icon="copyOutline"></ion-icon>
                Kopieren
              </ion-button>
              <ion-button fill="outline" size="small" @click="sharePayload">
                <ion-icon slot="start" :icon="shareOutline"></ion-icon>
                Teilen
              </ion-button>
            </div>
          </ion-card-content>
        </ion-card>

        <!-- Tag-Eigenschaften -->
        <ion-card>
          <ion-card-header>
            <ion-card-title>Tag-Eigenschaften</ion-card-title>
          </ion-card-header>
          <ion-card-content>
            <ion-item lines="none">
              <ion-label>
                <h3>Beschreibbar</h3>
                <p>{{ tagData.isWritable ? 'Ja' : 'Nein' }}</p>
              </ion-label>
              <ion-icon 
                slot="end" 
                :icon="tagData.isWritable ? checkmarkCircle : closeCircle"
                :color="tagData.isWritable ? 'success' : 'danger'"
              ></ion-icon>
            </ion-item>
            
            <ion-item lines="none" v-if="tagData.type !== 'Unbekannt'">
              <ion-label>
                <h3>Tag-Typ</h3>
                <p>{{ tagData.type }}</p>
              </ion-label>
            </ion-item>
            
            <ion-item lines="none" v-if="tagData.maxSize > 0">
              <ion-label>
                <h3>Maximale Größe</h3>
                <p>{{ tagData.maxSize }} Bytes</p>
              </ion-label>
            </ion-item>
            
            <ion-item lines="none" v-if="tagData.techTypes.length > 0">
              <ion-label>
                <h3>Technologien</h3>
                <p>{{ tagData.techTypes.join(', ') }}</p>
              </ion-label>
            </ion-item>
          </ion-card-content>
        </ion-card>

        <!-- Hex-Dump -->
        <ion-card v-if="tagData.allBytes.length > 0">
          <ion-card-header>
            <ion-card-title>Hex-Dump</ion-card-title>
          </ion-card-header>
          <ion-card-content>
            <div class="hex-dump">
              <pre>{{ formatHexDump(tagData.allBytes) }}</pre>
            </div>
          </ion-card-content>
        </ion-card>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButtons, IonBackButton,
  IonCard, IonCardHeader, IonCardTitle, IonCardContent, IonButton, IonIcon,
  IonItem, IonLabel, toastController
} from '@ionic/vue';
import { copyOutline, shareOutline, checkmarkCircle, closeCircle } from 'ionicons/icons';
import { Clipboard } from '@capacitor/clipboard';
import { Share } from '@capacitor/share';
import { useRoute } from 'vue-router';
import { onMounted, ref } from 'vue';

const route = useRoute();

// Tag-Daten Interface
interface TagData {
  id: string;
  payload: string;
  isWritable: boolean;
  idBytes: number[];
  payloadBytes: number[];
  techTypes: string[];
  maxSize: number;
  type: string;
  idHex: string;
  payloadHex: string;
  allBytes: number[];
}

const tagData = ref<TagData>({
  id: '',
  payload: '',
  isWritable: false,
  idBytes: [],
  payloadBytes: [],
  techTypes: [],
  maxSize: 0,
  type: '',
  idHex: '',
  payloadHex: '',
  allBytes: []
});

// Hilfsfunktionen
const bytesToHex = (bytes: number[]): string => {
  return bytes.map(b => b.toString(16).padStart(2, '0')).join(' ').toUpperCase();
};

const formatHexDump = (bytes: number[]): string => {
  let result = '';
  for (let i = 0; i < bytes.length; i += 16) {
    const chunk = bytes.slice(i, i + 16);
    const address = i.toString(16).padStart(8, '0').toUpperCase();
    const hex = chunk.map(b => b.toString(16).padStart(2, '0')).join(' ').toUpperCase().padEnd(47, ' ');
    const ascii = chunk.map(b => (b >= 32 && b <= 126) ? String.fromCharCode(b) : '.').join('');
    
    result += `${address}  ${hex}  |${ascii}|\n`;
  }
  return result;
};

// Route-Parameter verarbeiten
onMounted(() => {
  try {
    tagData.value = {
      id: route.params.id as string || 'Unbekannt',
      payload: decodeURIComponent(route.params.payload as string || ''),
      isWritable: route.params.isWritable === 'true',
      idBytes: JSON.parse(route.params.idBytes as string || '[]'),
      payloadBytes: JSON.parse(route.params.payloadBytes as string || '[]'),
      techTypes: JSON.parse(route.params.techTypes as string || '[]'),
      maxSize: parseInt(route.params.maxSize as string || '0'),
      type: route.params.type as string || 'Unbekannt',
      idHex: '',
      payloadHex: '',
      allBytes: []
    };

    // Hex-Darstellungen generieren
    if (tagData.value.idBytes.length > 0) {
      tagData.value.idHex = bytesToHex(tagData.value.idBytes);
    }
    
    if (tagData.value.payloadBytes.length > 0) {
      tagData.value.payloadHex = bytesToHex(tagData.value.payloadBytes);
    }

    // Alle Bytes für Hex-Dump kombinieren
    tagData.value.allBytes = [...tagData.value.idBytes, ...tagData.value.payloadBytes];

  } catch (error) {
    console.error('Fehler beim Verarbeiten der Tag-Daten:', error);
  }
});

// Payload in Zwischenablage kopieren
const copyToClipboard = async () => {
  try {
    await Clipboard.write({
      string: tagData.value.payload
    });

    const toast = await toastController.create({
      message: 'Payload in Zwischenablage kopiert',
      duration: 2000,
      position: 'bottom',
      color: 'success'
    });
    await toast.present();

  } catch (error) {
    console.error('Fehler beim Kopieren:', error);
    const toast = await toastController.create({
      message: 'Fehler beim Kopieren in die Zwischenablage',
      duration: 2000,
      position: 'bottom',
      color: 'danger'
    });
    await toast.present();
  }
};

// Payload teilen
const sharePayload = async () => {
  try {
    await Share.share({
      title: 'NFC-Tag Payload',
      text: tagData.value.payload,
      dialogTitle: 'NFC-Tag Payload teilen'
    });

  } catch (error) {
    console.error('Fehler beim Teilen:', error);
    const toast = await toastController.create({
      message: 'Fehler beim Teilen der Payload',
      duration: 2000,
      position: 'bottom',
      color: 'danger'
    });
    await toast.present();
  }
};
</script>

<style scoped>
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
  font-size: 12px;
  color: var(--ion-color-medium);
  word-break: break-all;
  margin: 4px 0;
}

.action-buttons {
  display: flex;
  gap: 8px;
  margin-top: 16px;
  flex-wrap: wrap;
}

.hex-dump {
  background-color: var(--ion-color-light);
  border-radius: 4px;
  padding: 12px;
  overflow-x: auto;
}

.hex-dump pre {
  font-family: 'Courier New', monospace;
  font-size: 12px;
  line-height: 1.4;
  margin: 0;
  white-space: pre;
  color: var(--ion-color-dark);
}

ion-card {
  margin: 8px 0;
}

ion-item {
  --padding-start: 0;
  --padding-end: 0;
}

ion-item h3 {
  margin: 0 0 4px 0;
  font-weight: 600;
}

ion-item p {
  margin: 0;
}
</style>
