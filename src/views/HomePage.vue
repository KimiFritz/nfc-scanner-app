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

      <!-- Buttons to read and write NFC tags -->
      <ion-button expand="block" @click="readNfc">Read</ion-button>
      <ion-button expand="block" @click="writeNfc">Write</ion-button>
      
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonButton } from '@ionic/vue';

const readNfc = () => {
};

const writeNfc = () => {
};
</script>

<style scoped>

#container {
  text-align: center;
  
  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}

#container p {
  font-size: 16px;
  line-height: 22px;
  
  color: #8c8c8c;
  
  margin: 0;
}

#container a {
  text-decoration: none;
}
</style>


