<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Barcode Scanner</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Barcode Scanner</ion-title>
        </ion-toolbar>
      </ion-header><br />
      <!-- Button to scan a barcode using the camera -->
      <ion-button expand="full" @click="scanBarcode()">Scan with camera</ion-button>
      <!-- Button to scan a barcode from an image in the gallery -->
      <ion-button expand="full" @click="scanBarcodeFromGallery()">Scan from Gallery</ion-button>
      <ion-item text-wrap></ion-item>

      <!-- List of scanned barcodes -->
      <ion-list>
        <ion-item-sliding v-for="(barcode, index) in reversedScannedBarcodes" :key="index">
          <ion-item>
            <ion-grid>
              <ion-row>
                <ion-col size="8" class="barcode-info">
                  <strong>Content:</strong><br />
                  {{ barcode.displayValue }}
                  <br />
                  <strong>Format:</strong> {{ barcode.format }}
                  <br />
                  <strong>Type:</strong> {{ barcode.valueType }}
                </ion-col>
                <ion-col size="4" class="barcode-value">
                  <ion-button v-if="barcode.valueType == 'URL' || barcode.valueType == 'PHONE'" @click="openOption(barcode)" color="primary" class="open-button">
                    <ion-icon slot="icon-only" :icon="openOutline" class="centered-icon"></ion-icon>
                  </ion-button>
                </ion-col>
              </ion-row>
            </ion-grid>
          </ion-item>
          <!-- Slide elements Copy, Share, and Delete -->
          <ion-item-options side="end">
            <ion-item-option @click="copyBarcode(index)" color="dark">
              <ion-icon slot="icon-only" :icon="copyIcon"></ion-icon>
              Copy
            </ion-item-option>
            <ion-item-option @click="shareBarcode(index)" color="primary">
              <ion-icon slot="icon-only" :icon="shareIcon"></ion-icon>
              Share
            </ion-item-option>
            <ion-item-option @click="deleteBarcode(reversedScannedBarcodes.length - index - 1)" color="danger">
              <ion-icon slot="icon-only" :icon="trashIcon"></ion-icon>
              Delete
            </ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonButtons,
  IonBackButton,
  IonItem,
  IonInput,
  IonButton,
  IonItemSliding,
  IonItemOption,
  IonItemOptions,
  IonIcon,
  IonCol,
  IonRow,
  IonGrid,
} from "@ionic/vue";
import { BarcodeScanner } from "@capacitor-mlkit/barcode-scanning";
import { FilePicker } from "@capawesome/capacitor-file-picker";
import { Preferences } from "@capacitor/preferences";
import { Share } from "@capacitor/share";
import { Clipboard } from "@capacitor/clipboard";
import { Browser } from "@capacitor/browser";
import { copy, trash, share, openOutline } from "ionicons/icons";
import { Toast } from "@capacitor/toast";

// Function to select images from the gallery
const pickImage = async () => {
  const { files } = await FilePicker.pickImages({});
  return files[0];
};

// Function to scan images from the gallery
const scanFromImage = async () => {
  const pickedImage = await pickImage();
  const { barcodes } = await BarcodeScanner.readBarcodesFromImage({
    path: pickedImage.path,
  });
  return barcodes[0];
};

// Function to request permissions
const requestPermissions = async () => {
  const { camera } = await BarcodeScanner.requestPermissions();
  return camera === "granted" || camera === "limited";
};

// Barcode interface
interface Barcode {
  displayValue: string;
  format: string;
  valueType: string;
}

export default defineComponent({
  components: {
    IonContent,
    IonHeader,
    IonPage,
    IonTitle,
    IonToolbar,
    IonButtons,
    IonBackButton,
    IonItem,
    IonInput,
    IonButton,
    IonItemSliding,
    IonItemOption,
    IonItemOptions,
    IonIcon,
    IonCol,
    IonRow,
    IonGrid,
  },

  data() {
    return {
      scannedBarcodes: [] as Barcode[],
      copyIcon: copy,
      trashIcon: trash,
      shareIcon: share,
      openOutline: openOutline,
      reversedList: [],
    };
  },

  computed: {
    reversedScannedBarcodes(): Barcode[] {
      return this.scannedBarcodes.slice().reverse();
    },
  },

  created() {
    this.loadScannedBarcodes();
  },

  methods: {
    // Function to scan a barcode using the camera, handling permissions and Play Services
    async scanBarcode() {
      const isAvailable = await BarcodeScanner.isGoogleBarcodeScannerModuleAvailable();
      if (!isAvailable.available) {
        try {
          await BarcodeScanner.installGoogleBarcodeScannerModule();
          await Toast.show({
            text: "Installing Google Play Services. Please wait a moment.",
            duration: "long",
          });
        } catch (error) {
          console.log("Failed to install Google Play Services:", error);
          await Toast.show({
            text: "Failed to install Google Play Services: " + error,
            duration: "long",
          });
        }
        const granted = await requestPermissions();
        if (!granted) {
          await Toast.show({
            text: "Allow camera access to scan codes.",
            duration: "long",
          });
          BarcodeScanner.openSettings();
          return;
        }
        const result = await BarcodeScanner.scan();
        const barcode = {
          displayValue: result.barcodes[0].displayValue,
          format: result.barcodes[0].format,
          valueType: result.barcodes[0].valueType,
        };
        this.scannedBarcodes.push(barcode);
        this.saveScannedBarcodes();
      } else {
        const granted = await requestPermissions();
        if (!granted) {
          await Toast.show({
            text: "Allow camera access to scan codes.",
            duration: "long",
          });
          BarcodeScanner.openSettings();
          return;
        }
        const result = await BarcodeScanner.scan();
        const barcode = {
          displayValue: result.barcodes[0].displayValue,
          format: result.barcodes[0].format,
          valueType: result.barcodes[0].valueType,
        };
        this.scannedBarcodes.push(barcode);
        this.saveScannedBarcodes();
      }
    },

    // Function to scan a barcode from an image in the gallery
    async scanBarcodeFromGallery() {
      const result = await scanFromImage();
      const barcode = {
        displayValue: result.displayValue,
        format: result.format,
        valueType: result.valueType,
      };
      this.scannedBarcodes.push(barcode);
      this.saveScannedBarcodes();
    },

    // Function to copy the content of a barcode to the clipboard
    async copyBarcode(index: number) {
      const barcode = this.reversedScannedBarcodes[index];
      await Clipboard.write({ string: barcode.displayValue });
    },

    // Function to delete a scanned barcode
    deleteBarcode(index: number) {
      this.scannedBarcodes.splice(index, 1);
      this.saveScannedBarcodes();
    },

    // Function to share a scanned barcode's content
    async shareBarcode(index: number) {
      const barcode = this.reversedScannedBarcodes[index];
      await Share.share({
        title: "Share Barcode",
        text: barcode.displayValue,
      });
    },

    // Function to open URLs or phone numbers from the scanned barcode
    async openOption(barcode: Barcode) {
      if (barcode.valueType === "URL") {
        await Browser.open({ url: barcode.displayValue });
      } else if (barcode.valueType === "PHONE") {
        await Browser.open({ url: `tel:${barcode.displayValue}` });
      }
    },

    // Function to save scanned barcodes to preferences
    async saveScannedBarcodes() {
      try {
        await Preferences.set({
          key: "scannedBarcodes",
          value: JSON.stringify(this.scannedBarcodes),
        });
      } catch (error) {
        console.error("Error saving scanned barcodes:", error);
      }
    },

    // Function to load previously scanned barcodes from preferences
    async loadScannedBarcodes() {
      try {
        const { value } = await Preferences.get({ key: "scannedBarcodes" });
        if (value) {
          this.scannedBarcodes = JSON.parse(value);
        }
      } catch (error) {
        console.error("Error loading scanned barcodes:", error);
      }
    },
  },
});
</script>

<style>
/* Container for the list */
.barcode-list {
  display: flex;
  flex-direction: column;
  align-items: center;
}
/* Styles for list entries */
.barcode-info {
  font-size: 16px;
  color: #fff;
}

/* Formatting for barcode values */
.barcode-value {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

/* Formatting for buttons to open URLs or phone numbers */
.barcode-value ion-button {
  --background: #007bff;
  --color: #fff;
  --padding-end: 0;
}

.barcode-value ion-icon {
  font-size: 24px;
}

.open-button {
  height: 50px;
  width: 50px;
}

.centered-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding-right: 9px;
}

/* Use black text color for light mode */
@media (prefers-color-scheme: light) {
  .barcode-info,
  .barcode-value ion-button {
    color: #000;
  }
}
</style>
