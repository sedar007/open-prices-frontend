<template>
  <v-card>
    <template #title>
      {{ $t('ReceiptAssistant.ReceiptItemsCheck') }}
    </template>

    <v-divider />

    <v-card-text>
      <v-data-table :headers="headers" :items="items" class="elevation-1" fixed-header hide-default-footer mobile-breakpoint="md" :mobile="null" items-per-page="100" :disable-sort="true">
        <template #[`item.product_name`]="{ item }">
          <v-text-field v-if="item.manuallyAdded" v-model="item.product_name" :hide-details="true" :rules="rules" dense single-line />
          <p v-else>
            {{ item.predicted_data.product_name }}
          </p>
          <p v-if="showInfoDetails">
            <span v-if="item.existingPrice" class="text-caption text-warning">{{ $t('ReceiptAssistant.PriceAlreadyCreatedForItem') }}</span>
            <span v-else-if="!item.predicted_data.price" class="text-caption text-error">{{ $t('ReceiptAssistant.MissingPrice') }}</span>
            <span v-else-if="item.isCategory ? !item.category_tag : !item.product_code" class="text-caption text-error">{{ $t('ReceiptAssistant.MissingProduct') }}</span>
            <span v-else class="text-caption text-success">{{ $t('ReceiptAssistant.PriceReadyToBeAdded') }}</span>
          </p>
        </template>
        <template #[`item.product`]="{ item }">
          <PriceCategoryChip v-if="item.isCategory" :priceCategory="item.category_tag" />
          <v-text-field 
            v-else-if="!item.productFound" 
            v-model="item.product_code" 
            :hide-details="true" 
            :rules="rules" 
            :append-inner-icon="item.product_code ? 'mdi-magnify' : 'mdi-barcode-scan'" 
            @click:append-inner="item.product_code ? findProduct(item) : launchBarcodeScanner(item)"
            @keydown.enter="findProduct(item)" 
          />
          <ProductCard v-else :product="item.productFound" :hideCategoriesAndLabels="true" :hideActionMenuButton="true" :readonly="true" elevation="1" />
        </template>
        <template #[`item.price`]="{ item }">
          <v-text-field v-model="item.predicted_data.price" :suffix="itemPriceSuffix(item)" :hide-details="true" :rules="rules" dense single-line />
        </template>
        <template #[`item.actions`]="{ item }">
          <v-row>
            <v-btn size="x-small" icon="mdi-pencil" @click="showEditProductDialog(item)" />
            <v-btn size="x-small" icon="mdi-delete" @click="deleteItem(item)" />
          </v-row>
        </template>
      </v-data-table>
    </v-card-text>

    <v-divider />

    <v-card-actions>
      <v-row>
        <v-col cols="6">
          <ProofReceiptPriceCountChip v-if="proof && items" class="mr-1" :uploadedCount="items.length" :totalCount="proof.receipt_price_count" />
          <ProofReceiptPriceTotalChip v-if="proof && items" :uploadedCount="proofPriceListSum" :totalCount="proof.receipt_price_total" :currency="proof.currency" />
        </v-col>
        <v-spacer />
        <v-col>
          <v-btn
            class="float-right"
            color="primary"
            variant="flat"
            @click="addItem"
          >
            {{ $t('ReceiptAssistant.AddNewItem') }}
          </v-btn>
        </v-col>
      </v-row>
    </v-card-actions>
  </v-card>

  <v-dialog v-model="editProductDialog" max-width="500px">
    <ContributionAssistantPriceFormCard
      v-if="editProductItem"
      :productPriceForm="editProductItem"
      :showProductNameField="true"
      :hideProductBarcodeScannerTab="false"
      :hideProofDetails="true"
      :hideActions="false"
      :hideUploadAction="false"
      :hidePriceTagStatusMenu="true"
      :isInDialog="true"
      forceMode="edit"
      @validatePriceTag="confirmProduct($event)"
      @close="editProductDialog = false"
    />
  </v-dialog>
  <BarcodeScannerDialog
    v-if="barcodeScannerDialog"
    v-model="barcodeScannerDialog"
    @barcode="setProductCodeFromScanner($event)"
    @close="barcodeScannerDialog = false"
  />
</template>
  
<script>
import { defineAsyncComponent } from 'vue'
import api from '../services/api'
import constants from '../constants'

export default {
  components: {
    ProductCard: defineAsyncComponent(() => import('../components/ProductCard.vue')),
    PriceCategoryChip: defineAsyncComponent(() => import('../components/PriceCategoryChip.vue')),
    ProofReceiptPriceCountChip: defineAsyncComponent(() => import('../components/ProofReceiptPriceCountChip.vue')),
    ProofReceiptPriceTotalChip: defineAsyncComponent(() => import('../components/ProofReceiptPriceTotalChip.vue')),
    ContributionAssistantPriceFormCard: defineAsyncComponent(() => import('../components/ContributionAssistantPriceFormCard.vue')),
    BarcodeScannerDialog: defineAsyncComponent(() => import('../components/BarcodeScannerDialog.vue')),
  },
  props: {
    proof: {
      type: Object,
      default: null
    },
    receiptItems: {
      type: Array,
      default: () => []
    },
    proofPriceExistingList: {
      type: Array,
      default: () => []
    }
  },
  emits: ['receiptItemsUpdated'],
  data() {
    return {
      showInfoDetails: true,
      items: [],
      headers: [
        { title: 'Product Name', key: 'product_name' },
        { title: 'Product', key: 'product' },
        { title: 'Price', key: 'price' },
        { title: 'Actions', key: 'actions' },
      ],
      editProductDialog: false,
      editProductItem: null,
      barcodeScannerDialog: false,
      barcodeScannerItem: null,
      rules: [
        v => !!v || '',
      ],
    }
  },
  computed: {
    proofPriceListSum() {
      return this.items.reduce((acc, price) => {
        return acc + parseFloat(price.predicted_data.price)
      }, 0)
    }
  },
  watch: {
    items: {
      handler(newReceiptItems) {
        this.$emit('receiptItemsUpdated', newReceiptItems)
      },
      deep: true
    },
    proof() {
      this.init()
    },
    proofPriceExistingList() {
      this.init()
    }
  }, 
  mounted() {
    this.init()
  },
  methods: {
    init() {
      if (!this.receiptItems?.length) return
      this.items = this.receiptItems.map((item) => {
        if (item.price_id) {
          item.existingPrice = this.proofPriceExistingList.find(price => price.id === item.price_id)
          item.productFound = item.existingPrice.product
          item.product_code = item.existingPrice.product?.code
          item.category_tag = item.existingPrice.category_tag
          item.isCategory = ![null, '', 'unknown', 'other'].includes(item.existingPrice.category_tag)
          item.price_per = item.existingPrice.price_per
          item.receipt_quantity = item.existingPrice.receipt_quantity
          // predictions
          item.predicted_data.price = item.existingPrice.price
          if (!item.predicted_data.product_name) {
            item.predicted_data.product_name = item.existingPrice.product_name
          }
        } else {
          item.productFound = null
          item.product_code = ""
          item.receipt_quantity = 1
          const categoryPredicted = ![null, '', 'unknown', 'other'].includes(item.predicted_data.product)
          if (categoryPredicted) {
            item.category_tag = item.predicted_data.product
            item.price_per = "KILOGRAM"
          }
        }
        return item
      })
    },
    itemPriceSuffix(item) {
      let suffix = this.proof.currency
      if (item.isCategory && item.category_tag) {
        suffix += '/' + (item.price_per === 'UNIT' ? 'U' : 'KG')
      }
      return suffix
    },
    findProduct(item) {
      api
        .getProductByCode(item.product_code)
        .then((data) => {
          const product = data.id ? data : {'code': item.product_code, 'price_count': 0}
          item.productFound = product
        })
        .catch((error) => {
          console.log(error)
          item.productFound = null
        })
    },
    deleteItem(item) {
      this.items.splice(this.items.indexOf(item), 1)
    },
    addItem() {
      this.items.push({
        manuallyAdded: true,
        product_code: '',
        product_name: '',
        price: null,
        receipt_quantity: 1,
        productFound: null,
        isCategory: false,
        category_tag: null,
        predicted_data: {}
      })
    },
    showEditProductDialog(item) {
      this.editProductDialog = true
      this.editProductItem = {
        item_id: this.items.indexOf(item),
        type: item.isCategory ? constants.PRICE_TYPE_CATEGORY : constants.PRICE_TYPE_PRODUCT,
        category_tag: ![null, '', 'unknown', 'other'].includes(item.category_tag) ? item.category_tag : null,
        origins_tags: [],
        labels_tags: [],
        price: item.price ? item.price.toString() : item.predicted_data.price.toString(),
        price_per: item.price_per,
        price_is_discounted: false,
        currency: this.proof.currency,
        receipt_quantity: item.receipt_quantity.toString(),
        proof: this.proof,
        proofImage: null,
        croppedImage: null,
        product_code: item.product_code,
        detected_product_code: item.product_code,
        product_name: item.predicted_data.product_name,
        loading: false
      }
    },
    confirmProduct() {
      this.editProductDialog = false
      this.items[this.editProductItem.item_id].productFound = this.editProductItem.product
      this.items[this.editProductItem.item_id].isCategory = this.editProductItem.type === constants.PRICE_TYPE_CATEGORY
      this.items[this.editProductItem.item_id].category_tag = this.editProductItem.type === constants.PRICE_TYPE_CATEGORY ? this.editProductItem.category_tag : null
      this.items[this.editProductItem.item_id].predicted_data.product_name = this.editProductItem.product_name
      this.items[this.editProductItem.item_id].predicted_data.price = this.editProductItem.price
      Object.assign(this.items[this.editProductItem.item_id], this.editProductItem)
      this.editProductItem = null
    },
    launchBarcodeScanner(item) {
      this.barcodeScannerDialog = true
      this.barcodeScannerItem = item
    },
    setProductCodeFromScanner(code) {
      this.barcodeScannerDialog = false
      this.barcodeScannerItem.product_code = code
      this.findProduct(this.barcodeScannerItem)
    }
  }
}
</script>

<style>
@media (max-width: 960px) {
  .v-table__wrapper > table > tbody > tr > td:first-child {
    padding-top: 40px !important;
  }
  .v-table__wrapper > table > tbody > tr > td:last-child {
    padding-bottom: 40px !important;
  }
  .v-table__wrapper > table > tbody > tr > td:last-child > div:first-child {
    height: 56px !important;
    align-content: center !important;
  }
  /**
  * hide "sort by" on mobile
  * https://stackoverflow.com/a/78435096
  */
  .v-data-table-headers--mobile {
    display: none;
  }
}
</style>
