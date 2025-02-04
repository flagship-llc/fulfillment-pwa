<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-back-button slot="start" default-href="/exim" />
        <ion-title>{{ $t("Download packed orders") }}</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <main>
        <ion-list>
          <ion-list-header>{{ $t("Select the fields you want to include in your export") }}</ion-list-header>
          <ion-button fill="clear" @click="selectAll" :disabled="!Object.keys(fieldMapping).length">{{ $t('Select all') }}</ion-button>

          <ion-item :key="field" v-for="(value, field) in fieldMapping">
            <ion-checkbox :checked="selectedData[field]" @click="isFieldClicked=true" @ionChange="updateSelectedData(field)" slot="start"/>
            <ion-label>{{ field }}</ion-label>
            <ion-button v-if="value === field" fill="outline" @click="addCustomLabel(field)">{{ $t('Custom Label') }}</ion-button>
            <!-- Using multiple if's instead of wrapping in a single parent div, to style the component properly without adding any extra css -->
            <ion-label v-if="value !== field" slot="end">{{ value }}</ion-label>
            <ion-button v-if="value !== field" slot="end" fill="clear" @click="addCustomLabel(field)">
              <ion-icon :icon="pencilOutline" />
            </ion-button>
          </ion-item>
        </ion-list>

        <ion-list>
          <ion-button fill="clear" @click="addCustomField()" :disabled="!Object.keys(fieldMapping).length">{{ $t('Add custom field') }}</ion-button>

          <ion-item :key="key" v-for="(value, key) in customFields">
            <ion-label>{{ key }}</ion-label>
            <ion-label slot="end">{{ value }}</ion-label>
            <ion-button slot="end" fill="clear" @click="removeCustomField(key)">
              <ion-icon :icon="trashOutline" />
            </ion-button>
          </ion-item>
        </ion-list>

        <ion-button size="large" :disabled="!content.length" color="medium" @click="download" expand="block">
          {{ $t("Download") }}
        </ion-button>

      </main>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { mapGetters } from "vuex";
import { alertController, IonBackButton, IonButton, IonCheckbox, IonContent, IonHeader, IonIcon, IonItem, IonLabel, IonList, IonListHeader, IonPage, IonTitle, IonToolbar, modalController } from '@ionic/vue'
import { pencilOutline, trashOutline } from 'ionicons/icons'
import { parseCsv, jsonToCsv, showToast } from '@/utils';
import { translate } from "@/i18n";
import logger from '@/logger';
import { DateTime } from 'luxon';
import { useRouter } from 'vue-router';
import { UploadService } from '@/services/UploadService';
import CustomFieldModal from '@/components/CustomFieldModal.vue'

export default defineComponent({
  name: 'UploadImportOrders',
  components: {
    IonBackButton,
    IonButton,
    IonCheckbox,
    IonContent,
    IonHeader,
    IonIcon,
    IonItem,
    IonLabel,
    IonList,
    IonListHeader,
    IonPage,
    IonTitle,
    IonToolbar
  },
  data() {
    return {
      content: [] as any,
      fieldMapping: {} as any,
      dataColumns: [] as Array<string>,
      selectedData: {} as any,
      isFieldClicked: false,
      customFields: {} as any
    }
  },
  computed: {
    ...mapGetters({
      currentFacility: 'user/getCurrentFacility'
    })
  },
  async ionViewDidEnter() {
    this.content = []
    await this.fetchPackedOrders();
  },
  methods: {
    async fetchPackedOrders() {
      // TODO: need to set a timeout, as when the orders size gets big then the api might take long time to return the information

      const payload = {
        params: {
          configId: 'MDM_PACKED_SHIPMENT',
          mimeTypeId: 'application/octet',
          facilityId: this.currentFacility.facilityId
        }
      }

      try {
        const resp = await UploadService.fetchPackedOrders(payload);

        if(resp.status == 200 && resp.data) {
          await this.parse(resp.data)
        } else {
          throw resp.data
        }
      } catch (err) {
        showToast(translate('Failed to get packed orders information'))
        logger.error('Failed to get packed orders', err)
      }
    },
    async parse(data: any) {
      try {
        this.content = await parseCsv(data).then(res => res);
        // get the column names from the data
        this.dataColumns = Object.keys(this.content[0]);
        // generate default mappings for the columns
        this.fieldMapping = this.dataColumns.reduce((fieldMapping: any, field: string) => {
          // check to not add the field for which the key is not available, as when fetching the data we are getting an empty key
          if(!field) {
            return fieldMapping;
          }

          fieldMapping[field] = field
          return fieldMapping;
        }, {})
      } catch {
        this.content = []
        logger.error("Failed to parse the data");
      }
    },
    async addCustomLabel(field: any) {
      const alert = await alertController.create({
        header: this.$t('Define custom label for', { field }),
        buttons: [{
          text: translate('Cancel'),
          role: 'cancel'
        }, {
          text: translate('Save'),
          handler: (data) => {
            const value = data.customLabel.trim();

            // check if the provided label is already mapped to some value, if yes then ask user to provide unique label
            if(Object.values(this.fieldMapping).includes(value) && this.fieldMapping[field] != value) {
              showToast(translate('Please provide unique labels'))
              return false;
            }

            this.fieldMapping[field] = value ? value : field;
            // once the field value is changed and if that field is already selected, then updating the data as well
            if(this.selectedData[field]) {
              this.selectedData[field] = this.fieldMapping[field]
            }
          }
        }],
        inputs: [{
          name: 'customLabel',
          type: 'text',
          placeholder: translate('Custom Label'),
          value: this.fieldMapping[field]
        }]
      });

      await alert.present();
    },
    updateSelectedData(field: any) {
      // Using isFieldClicked variable as when checked property is changed programatically it calls this method, resulting in wrong behaviour
      if(this.isFieldClicked) {
        if(this.selectedData[field]) {
          delete this.selectedData[field]
        } else {
          this.selectedData[field] = this.fieldMapping[field]
        }
        this.isFieldClicked = false;
      }
    },
    async download() {
      if(!Object.keys(this.selectedData).length) {
        showToast(translate('Please select at least one field to generate CSV'))
        return;
      }

      const downloadData = [] as any

      this.content.map((order: any) => {
        downloadData.push(Object.keys(this.selectedData).reduce((orderInfo: any, property: string) => {
          orderInfo[this.selectedData[property]] = order[property]
          return orderInfo
        }, {}))
      })

      // adding custom fields in the data
      Object.keys(this.customFields).map((field: any) => {
        downloadData.map((data: any) => data[field] = this.customFields[field])
      })

      const alert = await alertController.create({
        header: this.$t("Download packed orders"),
        message: this.$t("Make sure all the labels provided are correct."),
        buttons: [{
          text: this.$t("Cancel"),
          role: 'cancel',
        }, {
          text: this.$t("Download"),
          handler: async () => {
            const fileName = `HCPackedOrders-${this.currentFacility.facilityId}-${DateTime.now().toLocaleString(DateTime.DATETIME_MED_WITH_SECONDS)}.csv`
            await jsonToCsv(downloadData, { download: true, name: fileName })
            this.router.back();
          }
        }]
      });
      return alert.present();
    },
    selectAll() {
      this.selectedData = JSON.parse(JSON.stringify(this.fieldMapping))
    },
    async addCustomField() {
      const customFieldModal = await modalController.create({
        component: CustomFieldModal
      });

      customFieldModal.onDidDismiss().then((result) => {
        if(result.data && result.data.value) {
          this.customFields[result.data.value.key] = result.data.value.value
        }
      })

      return customFieldModal.present();
    },
    removeCustomField(key: any) {
      delete this.customFields[key];
    }
  },
  setup() {
    const router = useRouter();

    return {
      pencilOutline,
      trashOutline,
      router
    }
  }
});

</script>

<style scoped>
main {
  max-width: 732px;
  margin: var(--spacer-sm) auto 0;
}
</style>