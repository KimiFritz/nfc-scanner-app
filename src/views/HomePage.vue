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
        <p :class="['status-message', `status-${nfcStatus}`]">
          {{ statusMessage }}
        </p>
      </div>

      <!-- Buttons to read and write NFC tags -->
      <div id="container">
        <ion-button 
          expand="block" 
          @click="readNfc"
          :disabled="!isNfcReady"
        >
          Read NFC Tag
        </ion-button>
        <ion-button 
          expand="block" 
          @click="writeNfc"
          :disabled="!isNfcReady"
        >
          Write NFC Tag
        </ion-button>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton } from '@ionic/vue';
import { Nfc, NfcUtils, NfcTagTechType} from '@capawesome-team/capacitor-nfc';
import { alertController } from '@ionic/vue';
import { Capacitor } from '@capacitor/core';
import { onMounted, ref } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();

// Reactive state für NFC-Status
const nfcStatus = ref<'checking' | 'ready' | 'error'>('checking');
const statusMessage = ref('NFC wird initialisiert...');
const isNfcReady = ref(false);

// App-Start: Vollständige NFC-Initialisierung
onMounted(async () => {
  await initializeNfc();
});

const initializeNfc = async () => {
  try {
    nfcStatus.value = 'checking';
    statusMessage.value = 'NFC wird geprüft...';

    // 1. NFC-Support prüfen
    const { isSupported } = await Nfc.isSupported();
    if (!isSupported) {
      nfcStatus.value = 'error';
      statusMessage.value = '❌ NFC wird von diesem Gerät nicht unterstützt';
      await showErrorDialog('NFC nicht verfügbar', 'Diese App benötigt NFC-Unterstützung.');
      return;
    }

    // 2. NFC-Aktivierung prüfen (nur Android)
    if (Capacitor.getPlatform() === 'android') {
      const { isEnabled } = await Nfc.isEnabled();
      if (!isEnabled) {
        nfcStatus.value = 'error';
        statusMessage.value = '📱 NFC ist deaktiviert';
        await promptNfcActivation();
        return;
      }
    }

    // 3. Berechtigungen prüfen und anfordern
    statusMessage.value = 'Berechtigungen werden geprüft...';
    const { nfc } = await Nfc.checkPermissions();
    
    if (nfc !== 'granted') {
      statusMessage.value = 'Berechtigung wird angefordert...';
      const { nfc: requestedPermission } = await Nfc.requestPermissions();
      
      if (requestedPermission !== 'granted') {
        nfcStatus.value = 'error';
        statusMessage.value = '❌ NFC-Berechtigung verweigert';
        await showErrorDialog('Berechtigung erforderlich', 'Diese App benötigt NFC-Berechtigung.');
        return;
      }
    }

    // 4. Erfolgreich - App ist bereit
    nfcStatus.value = 'ready';
    statusMessage.value = '✅ NFC ist bereit';
    isNfcReady.value = true;

  } catch (error) {
    console.error('Fehler bei NFC-Initialisierung:', error);
    nfcStatus.value = 'error';
    statusMessage.value = '❌ Fehler bei der NFC-Initialisierung';
    await showErrorDialog('Initialisierungsfehler', `Fehler: ${(error as Error).message}`);
  }
};

const promptNfcActivation = async () => {
  const alert = await alertController.create({
    header: 'NFC aktivieren',
    message: 'NFC ist deaktiviert. Bitte aktivieren Sie NFC in den Einstellungen.',
    buttons: [
      { text: 'Abbrechen', role: 'cancel' },
      { 
        text: 'Einstellungen öffnen', 
        handler: async () => {
          await Nfc.openSettings();
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
    buttons: [
      { text: 'Erneut versuchen', handler: () => initializeNfc() },
      { text: 'OK', role: 'cancel' }
    ]
  });
  await alert.present();
};

// Hilfsfunktionen für NFC-Datenkonvertierung
const bytesToHex = (bytes: number[]): string => {
  return bytes.map(b => b.toString(16).padStart(2, '0')).join(' ').toUpperCase();
};

const bytesToString = (bytes: number[]): string => {
  try {
    return new TextDecoder('utf-8').decode(new Uint8Array(bytes));
  } catch {
    return bytesToHex(bytes); // Fallback to hex if not valid UTF-8
  }
};

// NFC-Lesefunktion mit Alert-Dialog
const readNfc = async () => {
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
      
      // Tag-Daten extrahieren
      const tagData = {
        id: tag.id ? bytesToHex(tag.id) : 'Unbekannt',
        idBytes: tag.id || [],
        payload: '',
        payloadBytes: [] as number[],
        isWritable: tag.isWritable || false,
        techTypes: tag.techTypes || [],
        maxSize: tag.maxSize || 0,
        type: tag.type || 'Unbekannt'
      };

      // Payload aus NDEF-Records extrahieren
      if (tag.message?.records && tag.message.records.length > 0) {
        const firstRecord = tag.message.records[0];
        if (firstRecord.payload) {
          tagData.payloadBytes = firstRecord.payload;
          tagData.payload = bytesToString(firstRecord.payload);
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
          payloadBytes: JSON.stringify(tagData.payloadBytes),
          techTypes: JSON.stringify(tagData.techTypes),
          maxSize: tagData.maxSize.toString(),
          type: tagData.type
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
  padding: 20px;
  margin-top: 20px;
}

.status-message {
  font-size: 16px;
  font-weight: 500;
  margin: 0;
  padding: 12px 20px;
  border-radius: 8px;
  transition: all 0.3s ease;
}

.status-checking {
  background-color: #e3f2fd;
  color: #1565c0;
  border: 1px solid #bbdefb;
}

.status-ready {
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.status-error {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

#container {
  text-align: center;
  position: absolute;
  left: 0;
  right: 0;
  top: 60%;
  transform: translateY(-50%);
  padding: 20px;
}

#container ion-button {
  margin: 10px 0;
}

#container ion-button[disabled] {
  opacity: 0.4;
}
</style>
