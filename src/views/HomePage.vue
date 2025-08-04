<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>NFC-Scanner-App</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">NFC-Scanner-App</ion-title>
        </ion-toolbar>
      </ion-header>

      <!-- NFC Status Anzeige -->
      <div id="status-container">
        <div :class="['status-card', `status-${nfcStatus}`]">
          <ion-icon 
            :icon="getStatusIcon()" 
            :class="['status-icon', `status-icon-${nfcStatus}`]"
          ></ion-icon>
          <p class="status-message">{{ statusMessage }}</p>
        </div>
      </div>

      <!-- NFC Icon in der Mitte -->
      <div id="nfc-icon-container">
        <ion-icon :icon="radioOutline" class="nfc-main-icon"></ion-icon>
      </div>

      <!-- Buttons to read and write NFC tags -->
      <div id="container">
        <ion-button 
          expand="block" 
          @click="readNfc"
          :disabled="!isNfcReady"
        >
          <ion-icon :icon="scanOutline" slot="start"></ion-icon>
          Lesen
        </ion-button>
        <ion-button 
          expand="block" 
          @click="writeNfc"
          :disabled="!isNfcReady"
        >
          <ion-icon :icon="createOutline" slot="start"></ion-icon>
          Schreiben
        </ion-button>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton, IonIcon } from '@ionic/vue';
import { scanOutline, createOutline, radioOutline, checkmarkCircleOutline, alertCircleOutline, timeOutline, settingsOutline } from 'ionicons/icons';
import { Nfc, NfcUtils } from '@capawesome-team/capacitor-nfc';
import { alertController } from '@ionic/vue';
import { Capacitor } from '@capacitor/core';
import { onMounted, ref } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();

// Reactive state für NFC-Status
const nfcStatus = ref<'checking' | 'ready' | 'error'>('checking');
const statusMessage = ref('NFC wird geprüft...');
const isNfcReady = ref(false);

// Status Icon bestimmen
const getStatusIcon = () => {
  switch (nfcStatus.value) {
    case 'checking': return timeOutline;
    case 'ready': return checkmarkCircleOutline;
    case 'error': return alertCircleOutline;
    default: return timeOutline;
  }
};

// App-Start: Nur grundlegende NFC-Verfügbarkeit prüfen
onMounted(async () => {
  await checkNfcAvailability();
});

const checkNfcAvailability = async () => {
  try {
    nfcStatus.value = 'checking';
    statusMessage.value = 'NFC wird geprüft...';

    // 1. NFC-Support prüfen
    const { isSupported } = await Nfc.isSupported();
    if (!isSupported) {
      nfcStatus.value = 'error';
      statusMessage.value = 'NFC wird nicht unterstützt';
      return;
    }

    // 2. NFC-Aktivierung prüfen (nur Android)
    if (Capacitor.getPlatform() === 'android') {
      const { isEnabled } = await Nfc.isEnabled();
      if (!isEnabled) {
        nfcStatus.value = 'error';
        statusMessage.value = 'NFC ist deaktiviert';
        await showNfcDeactivationDialog();
        return;
      }
    }

    // 3. NFC ist verfügbar und aktiviert
    nfcStatus.value = 'ready';
    statusMessage.value = 'NFC ist bereit';
    isNfcReady.value = true;

  } catch (error) {
    console.error('Fehler bei NFC-Prüfung:', error);
    nfcStatus.value = 'error';
    statusMessage.value = 'Fehler bei der NFC-Prüfung';
  }
};

const showNfcDeactivationDialog = async () => {
  const alert = await alertController.create({
    header: 'NFC aktivieren',
    message: 'NFC ist auf diesem Gerät deaktiviert. Bitte aktivieren Sie NFC in den Einstellungen, um die App verwenden zu können.',
    buttons: [
      { 
        text: 'Abbrechen', 
        role: 'cancel',
        handler: () => {
          nfcStatus.value = 'error';
          statusMessage.value = 'NFC ist deaktiviert';
        }
      },
      { 
        text: 'Einstellungen öffnen', 
        handler: async () => {
          await Nfc.openSettings();
          // Nach dem Öffnen der Einstellungen erneut prüfen
          setTimeout(() => checkNfcAvailability(), 1000);
        }
      }
    ]
  });
  await alert.present();
};

const checkPermissionsAndRequest = async (): Promise<boolean> => {
  try {
    // Berechtigungen prüfen
    const { nfc } = await Nfc.checkPermissions();
    
    if (nfc !== 'granted') {
      // Berechtigung anfordern
      const { nfc: requestedPermission } = await Nfc.requestPermissions();
      
      if (requestedPermission !== 'granted') {
        await showPermissionDialog();
        return false;
      }
    }
    
    return true;
  } catch (error) {
    console.error('Fehler bei Berechtigungsprüfung:', error);
    await showErrorDialog('Berechtigungsfehler', `Fehler: ${(error as Error).message}`);
    return false;
  }
};

const showPermissionDialog = async () => {
  const alert = await alertController.create({
    header: 'NFC-Berechtigung erforderlich',
    message: 'Diese App benötigt NFC-Berechtigung zum Lesen und Schreiben von Tags.',
    buttons: [
      { text: 'Abbrechen', role: 'cancel' },
      { 
        text: 'Einstellungen öffnen', 
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

const showErrorDialog = async (header: string, message: string) => {
  const alert = await alertController.create({
    header,
    message,
    buttons: ['OK']
  });
  await alert.present();
};

// NFC-Lesefunktion mit Berechtigungsprüfung
const readNfc = async () => {
  // Erst Berechtigungen prüfen
  const hasPermission = await checkPermissionsAndRequest();
  if (!hasPermission) return;

  let scanAlert: any = null;
  let tagListener: any = null;
  let errorListener: any = null;
  let cancelListener: any = null;

  // Zentrale Cleanup-Funktion
  const cleanup = async () => {
    try {
      await Nfc.stopScanSession();
      await scanAlert?.dismiss();
      tagListener?.remove();
      errorListener?.remove();
      cancelListener?.remove();
    } catch (error) {
      console.error('Fehler beim Cleanup:', error);
    }
  };

  try {
    // Alte Listener entfernen
    await Nfc.removeAllListeners();

    // Scan-Dialog erstellen
    scanAlert = await alertController.create({
      header: 'NFC-Tag scannen',
      message: 'Halten Sie ein NFC-Tag an das Gerät',
      backdropDismiss: false,
      buttons: [
        {
          text: 'Abbrechen',
          role: 'cancel',
          handler: cleanup
        }
      ]
    });

    await scanAlert.present();

    // Tag-Listener
    tagListener = await Nfc.addListener('nfcTagScanned', async (event) => {
      await cleanup();

      const tag = event.nfcTag;
      const utils = new NfcUtils();
      
      // Tag-Daten extrahieren
      const tagData = {
        id: tag.id ? utils.convertBytesToHex({ bytes: tag.id }).hex : 'Unbekannt',
        idBytes: tag.id || [],
        payload: '',
        payloadBytes: [] as number[],
        isWritable: tag.isWritable || false
      };

      // Payload aus NDEF-Records extrahieren (nur Text-Records)
      if (tag.message?.records && tag.message.records.length > 0) {
        const firstRecord = tag.message.records[0];
        if (firstRecord.payload) {
          tagData.payloadBytes = firstRecord.payload;
          
          // Versuche Text aus NDEF Text Record zu extrahieren
          try {
            const { text } = utils.getTextFromNdefTextRecord({ record: firstRecord });
            if (text) {
              tagData.payload = text;
            } else {
              // Fallback: Bytes als String interpretieren
              tagData.payload = utils.convertBytesToString({ bytes: firstRecord.payload }).text;
            }
          } catch {
            // Fallback: Bytes als String interpretieren
            tagData.payload = utils.convertBytesToString({ bytes: firstRecord.payload }).text;
          }
        }
      }

      // Zur Detailseite navigieren
      router.push({
        name: 'nfc-detail',
        params: {
          id: tagData.id,
          payload: encodeURIComponent(tagData.payload),
          isWritable: tagData.isWritable.toString(),
          idBytes: JSON.stringify(tagData.idBytes),
          payloadBytes: JSON.stringify(tagData.payloadBytes)
        }
      });
    });

    // Error-Listener
    errorListener = await Nfc.addListener('scanSessionError', async (event) => {
      await cleanup();
      
      const errorAlert = await alertController.create({
        header: 'Scan-Fehler',
        message: event.message || 'Fehler beim Scannen des NFC-Tags.',
        buttons: ['OK']
      });
      await errorAlert.present();
    });

    // Cancel-Listener für iOS
    cancelListener = await Nfc.addListener('scanSessionCanceled', cleanup);

    // Scan-Session starten
    await Nfc.startScanSession({
      alertMessage: 'NFC-Tag an das Gerät halten'
    });

  } catch (error) {
    await cleanup();
    
    console.error('Fehler beim NFC-Scan:', error);
    const errorAlert = await alertController.create({
      header: 'Fehler',
      message: 'Ein Fehler ist beim NFC-Scan aufgetreten.',
      buttons: ['OK']
    });
    await errorAlert.present();
  }
};

const write = async (text: string) => {
  const utils = new NfcUtils();
  const { record } = utils.createNdefTextRecord({ text });

  return new Promise<void>(async (resolve, reject) => {
    let cancelled = false;

    const writingAlert = await alertController.create({
      header: 'Warte auf NFC-Tag...',
      message: 'Halte das NFC-Tag an das Gerät, um den Text zu schreiben.',
      buttons: [
        {
          text: 'Abbrechen',
          role: 'cancel',
          handler: async () => {
            cancelled = true;
            await Nfc.stopScanSession();
            resolve();
          }
        }
      ],
      backdropDismiss: false,
    });
    await writingAlert.present();

    await Nfc.removeAllListeners();

    const listener = await Nfc.addListener('nfcTagScanned', async () => {
      if (cancelled) return;

      try {
        await Nfc.write({ message: { records: [record] } });
        await Nfc.stopScanSession();
        await writingAlert.dismiss();

        const successAlert = await alertController.create({
          header: 'Erfolg',
          message: 'Der Text wurde auf das NFC-Tag geschrieben.',
          buttons: ['OK']
        });
        await successAlert.present();

        resolve();
      } catch (error) {
        await writingAlert.dismiss();

        const errorAlert = await alertController.create({
          header: 'Fehler',
          message: 'Fehler beim Schreiben auf das NFC-Tag.',
          buttons: ['OK']
        });
        await errorAlert.present();

        reject(error);
      }
    });

    await Nfc.startScanSession({ alertMessage: 'Halte das NFC-Tag zum Schreiben an das Gerät.' });
  });
};

const writeNfc = async () => {
  // Erst Berechtigungen prüfen
  const hasPermission = await checkPermissionsAndRequest();
  if (!hasPermission) return;

  const alert = await alertController.create({
    header: 'NFC-Tag beschreiben',
    message: 'Bitte gib den Text ein, der auf das NFC-Tag geschrieben werden soll:',
    inputs: [
      {
        name: 'nfcText',
        type: 'text',
        placeholder: 'Text für NFC-Tag',
      }
    ],
    buttons: [
      {
        text: 'Abbrechen',
        role: 'cancel',
      },
      {
        text: 'OK',
        handler: async (data: { nfcText: string }) => {
          if (data.nfcText && data.nfcText.trim().length > 0) {
            // Kleiner Delay, damit der Alert sauber schließt
            setTimeout(() => write(data.nfcText), 300);
          }
        }
      }
    ]
  });
  await alert.present();
};
</script>


<style scoped>
#status-container {
  text-align: center;
  padding: 16px;
  margin-top: 20px;
}

.status-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 16px 20px;
  border-radius: 12px;
  transition: all 0.3s ease;
  max-width: 300px;
  margin: 0 auto;
}

.status-icon {
  font-size: 32px;
  margin-bottom: 8px;
}

.status-icon-checking {
  color: #1565c0;
}

.status-icon-ready {
  color: #2e7d32;
}

.status-icon-error {
  color: #d32f2f;
}

.status-message {
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  text-align: center;
}

.status-checking {
  background-color: #e3f2fd;
  color: #1565c0;
  border: 1px solid #bbdefb;
}

.status-ready {
  background-color: #e8f5e8;
  color: #2e7d32;
  border: 1px solid #c8e6c9;
}

.status-error {
  background-color: #ffebee;
  color: #d32f2f;
  border: 1px solid #ffcdd2;
}

#nfc-icon-container {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 40px 0;
}

.nfc-main-icon {
  font-size: 80px;
  color: var(--ion-color-primary);
  opacity: 0.7;
}

#container {
  text-align: center;
  position: fixed;
  left: 0;
  right: 0;
  bottom: 80px;
  padding: 20px;
}

#container ion-button {
  margin: 10px 0;
}

#container ion-button[disabled] {
  opacity: 0.4;
}
</style>
