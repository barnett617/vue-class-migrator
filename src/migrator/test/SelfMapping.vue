<template>
  <div class="section self-mapping">
    <p class="section-question">
      {{ $t("pAvailability_cSelfMapping_MainQuestion") }}
    </p>
    <p class="instruction">
      {{ $t("pAvailability_cSelfMapping_Instruction") }}
    </p>

    <RadioSelectionGroup
      :selected="selfMapping.hasSelfMapping"
      name="has-self-mapping"
      class="self-mapping-selection"
      @click="onHasSelfMappingSelection"
    />

    <BaseAlert
      v-show="selfMapping.hasSelfMapping === false"
      :message="
        $t('pAvailability_cSelfMapping_NoBookingSystemAlert', [
          '<a href=\'https://getconnected.getyourguide.com/\' target=\'_blank\'>',
          '</a>',
        ])
      "
      class="self-mapping-alert"
    />

    <div v-if="selfMapping.hasSelfMapping" class="self-mapping-form indented-section">
      <div class="self-mapping-form-item">
        <label class="input-label is-bold">
          {{ $t("pAvailability_cSelfMapping_FormTitle") }}
        </label>
        <BaseSelect
          :default-option="$t('pAvailability_cSelfMapping_SelectDefault')"
          :options="selfMapping.availableReservationSystems.value"
          size="large"
          :selected="selfMapping.reservationSystem.value"
          @change="onReservationSystemSelection"
        />
      </div>

      <div
        v-show="selfMapping.reservationSystem.value && hasSupportedSystem"
        class="self-mapping-form-item"
      >
        <label class="input-label is-bold">
          {{ $t("pAvailability_cSelfMapping_productId") }}
        </label>
        <BaseTextField
          :has-counter="false"
          :required="true"
          size="large"
          :value="selfMapping.externalProductId.value"
          :name="selfMapping.externalProductId.name"
          @change="onProductIdInput"
        />
      </div>

      <div v-show="!hasSupportedSystem" class="self-mapping-form-item">
        <label class="input-label is-bold">
          {{ $t("pAvailability_cSelfMapping_CustomSystem") }}
        </label>
        <BaseTextField
          :has-counter="false"
          :required="true"
          size="large"
          :value="selfMapping.customSystem.value"
          :name="selfMapping.customSystem.name"
          @change="onCustomReservationSystemInput"
        />
      </div>

      <div class="section connectivity-type">
        <p class="section-question">
          {{ $t("pAvailability_cSelfMapping_ConnectivityTypeQuestion") }}
        </p>
        <RadioSelectionGroup
          :items="connectivityTypes"
          :selected="connectivityTypeSelector"
          :required="true"
          name="connectivity-type"
          @click="onConnectivityTypeSelection"
        />
      </div>

      <div class="save-button-container">
        <button
          v-if="selfMapping.errors.length"
          type="button"
          class="btn btn-small btn-ghost"
          @click="onContinueManuallyClick"
        >
          {{ $t("pAvailability_cSelfMapping_ContinueManually") }}
        </button>
      </div>

      <SelfMappingResultDisplay
        :self-mapping="selfMapping"
        :supplier-id="supplierId"
        :company-name="companyName"
        :option-id="optionId"
        :option-title="optionTitle"
        :connectivity-type="connectivityType"
        :has-supported-system="hasSupportedSystem"
        :show-success-banner="showSuccessBanner"
      />
    </div>

    <c-modal v-if="showPricesOverApiModal" @close="closePricesOverApiModal">
      <div class="instructions">
        <p class="instructions_header">
          {{ $t("pPricing_cSelfMapping_PricesOverApiModalTitle") }}
        </p>
        <ul class="sub-instruction">
          <li class="instruction">
            {{ $t("pPricing_cSelfMapping_PricesOverApiModalReservationSystem") }}
          </li>
          <li class="instruction">
            {{ $t("pPricing_cSelfMapping_PricesOverApiModalFirstPoint") }}
          </li>
          <li class="instruction">
            {{ $t("pPricing_cSelfMapping_PricesOverApiModalSecondPoint") }}
          </li>
        </ul>
        <p class="instructions_footer">
          {{ $t("pPricing_cSelfMapping_PricesOverApiModalFooter") }}
        </p>
      </div>

      <template #leftAction>
        <span></span>
      </template>
      <template #footer>
        <c-button variant="text" @click="closePricesOverApiModal">
          {{ $t("pPricing_cSelfMapping_CancelPricesOverApi") }}
        </c-button>
        <c-button color="critical" @click="confirmPricesOverApiModal">
          {{ $t("pPricing_cSelfMapping_SavePricesOverApi") }}
        </c-button>
      </template>
    </c-modal>

    <c-modal v-if="showChangingFromPricesOverApiModal" @close="closePricesOverApiModal">
      <p>{{ $t("pPricing_cSelfMapping_Availability_Only_Modal_Header") }}</p>
      <ul class="availability-instructions">
        <li class="instruction">
          {{ $t("pPricing_cSelfMapping_Availability_Only_Modal_First_Instruction") }}
        </li>
        <li class="instruction">
          {{ $t("pPricing_cSelfMapping_Availability_Only_Modal_Second_Instruction") }}
        </li>
      </ul>
      <p>{{ $t("pPricing_cSelfMapping_Availability_Only_Modal_Footer") }}</p>
      <template #leftAction>
        <span></span>
      </template>
      <template #footer>
        <c-button variant="text" @click="closeCancellingPricesOverApiModal">
          {{ $t("pPricing_cSelfMapping_CancelPricesOverApi") }}
        </c-button>
        <c-button color="critical" @click="confirmCancellingPricesOverApi">
          {{ $t("pPricing_cSelfMapping_SavePricesOverApi") }}
        </c-button>
      </template>
    </c-modal>
  </div>
</template>

<script lang="ts">
// @ts-nocheck

import RadioSelectionGroup from "@Components/RadioSelectionGroup.vue";
import { BaseSelect, BaseTextField } from "@Components/Inputs";
import { CButton, CModal } from "@getyourguide/compass";
import BaseAlert from "@Components/BaseAlert.vue";
import isEmpty from "lodash/isEmpty";
import ErrorsList from "@Components/ErrorsList.vue";
import { Component, Prop, Vue, Watch } from "vue-property-decorator";
import SelfMappingResultDisplay from "./SelfMappingResultDisplay.vue";

const UNSUPPORTED_SYSTEM_KEY = "another";
const NOT_CONNECTED = "not_connected";

@Component({
  components: {
    ErrorsList,
    CButton,
    BaseTextField,
    BaseSelect,
    RadioSelectionGroup,
    BaseAlert,
    CModal,
    SelfMappingResultDisplay,
  },
})
export default class SelfMapping extends Vue {
  @Prop({ type: Number, required: true }) readonly tourId!: number;
  @Prop({ type: String, required: true }) readonly optionTitle!: string;
  @Prop({ type: Object, required: true }) readonly selfMapping!: {
    errors: string[];
    isValid: boolean;
    hasSelfMapping: boolean;
    availableReservationSystems: {
      value: [];
    };
    externalProductId: {
      value: number;
      name: string;
    };
    customSystem: {
      value: string;
      name: string;
    };
    reservationSystem: {
      value: string;
    };
  };
  @Prop({ type: Boolean, required: true }) readonly isStaff!: boolean;
  @Prop({ type: Number, required: true }) readonly supplierId!: number;
  @Prop({ type: String, required: true }) readonly companyName!: string;
  @Prop({ type: Object, required: true })
  readonly connectivityTypes!: Record<string, unknown>;
  @Prop({ required: true }) readonly connectivityType!: string | null;
  @Prop({ type: Boolean, default: false, required: true }) readonly showSuccessBanner!: boolean;

  availableReservationSystems = {};
  optionId = this.$store.state.optionId;
  showPricesOverApiModal = false;
  showChangingFromPricesOverApiModal = false;

  @Watch("disableSubmission", { immediate: true })
  allowSelfMappingSave(): void {
    this.$emit("saveSelfMappingEnabled", !this.disableSubmission);
  }

  get connectivityTypeSelector(): string {
    // Used to reset radio selection after modal selection
    return this.showPricesOverApiModal ? "" : this.connectivityType;
  }

  get hasSupportedSystem() {
    return (
      this.selfMapping.reservationSystem.value !== UNSUPPORTED_SYSTEM_KEY &&
      this.selfMapping.reservationSystem.value !== NOT_CONNECTED
    );
  }

  get disableSubmission(): boolean {
    if (isEmpty(this.selfMapping.reservationSystem.value)) {
      return true;
    }

    if (this.hasSupportedSystem && isEmpty(this.selfMapping.externalProductId.value)) {
      return true;
    }

    if (this.selfMapping.isValid) {
      return true;
    }

    return !this.hasSupportedSystem && isEmpty(this.selfMapping.customSystem.value);
  }

  get hasPricesOverApi(): boolean {
    return (
      this.$store.state.pricing.selfMapping.connectivityType.value === "availability_and_price"
    );
  }

  async onHasSelfMappingSelection(hasSelfMapping: boolean): Promise<void> {
    let { tourId, optionId, hasPricesOverApi } = this;

    if (hasPricesOverApi && !hasSelfMapping) {
      this.showChangingFromPricesOverApiModal = true;

      this.$store.commit("pricing/SET_HAS_SELF_MAPPING", false);
      this.resetConnectivityType();

      this.setPageSyncAndSaved(false);

      // Don't update Fishfarm yet
      return;
    }

    // reset in case user ran through the form before
    this.resetConnectivityType();

    this.setPageSyncAndSaved(false);
    this.$store.dispatch("pricing/setHasSelfMapping", {
      hasSelfMapping,
      tourId,
      optionId,
    });

    this.$analyticsLogger.log("SupplierWebUIClick", {
      target: "SetIsConnected",
      id: "SetIsConnected",
      metadata: {
        is_connected: hasSelfMapping,
        tour_id: tourId,
        option_id: optionId,
      },
    });
  }

  async setPageSyncAndSaved(status: boolean): Promise<void> {
    await this.$store.dispatch("pricing/setIsSynced", status);
    await this.$store.dispatch("pricing/setIsSaved", status);
  }

  resetConnectivityType(): void {
    this.$store.dispatch("pricing/setConnectivityType", "");
  }

  resetPricingType(): void {
    this.setPricingType(null);
  }

  async onConnectivityTypeSelection(value: string): Promise<void> {
    if (value === "availability_and_price") {
      this.showPricesOverApiModal = true;
      return;
    } else if (this.hasPricesOverApi && value !== "availability_and_price") {
      await this.$store.dispatch("pricing/setConnectivityType", value);
      this.showChangingFromPricesOverApiModal = true;
    } else {
      this.$store.dispatch("pricing/setConnectivityType", value);
    }

    this.sendConnectivityTypeEvent(false);
  }

  async setPricingType(type: string | null): Promise<void> {
    let { tourId, optionId } = this;
    await this.$store.dispatch("pricing/setPricingType", { tourId, optionId, type });
  }

  onReservationSystemSelection(target) {
    const { value } = target;

    this.$store.dispatch("pricing/setReservationSystem", value);
    this.setPageSyncAndSaved(false);
  }

  async onProductIdInput(target) {
    const { value } = target;

    this.$store.dispatch("pricing/setReservationSystemProductId", value);
    this.setPageSyncAndSaved(false);
  }

  async onCustomReservationSystemInput(target) {
    const { value } = target;

    this.$store.dispatch("pricing/setCustomReservationSystem", value);
    this.setPageSyncAndSaved(false);
  }

  async sendConnectivityTypeEvent(isPricesOverApi: boolean) {
    this.$analyticsLogger.log("SupplierWebUIClick", {
      target: "IsPricesOverApi",
      id: "IsPricesOverApi",
      metadata: {
        is_prices_over_api: isPricesOverApi,
        tour_id: this.tourId,
        option_id: this.optionId,
      },
    });
  }

  async confirmPricesOverApiModal(): Promise<void> {
    this.showPricesOverApiModal = false;
    await this.$store.dispatch("pricing/setConnectivityType", "availability_and_price");
    this.sendConnectivityTypeEvent(true);
    this.$store.commit("pricing/RESET_PRICINGS");
  }

  closePricesOverApiModal(): void {
    this.$store.dispatch("pricing/setConnectivityType", "availability_only");
    this.showPricesOverApiModal = false;
  }

  async confirmCancellingPricesOverApi(): Promise<void> {
    const { tourId, optionId, selfMapping } = this;
    this.showChangingFromPricesOverApiModal = false;
    await this.$store.dispatch("pricing/setHasSelfMapping", {
      hasSelfMapping: selfMapping.hasSelfMapping,
      tourId,
      optionId,
    });

    this.$store.commit("pricing/RESET_PRICINGS");
  }

  async closeCancellingPricesOverApiModal(): Promise<void> {
    const { tourId, optionId } = this;
    await this.$store.dispatch("pricing/setHasSelfMapping", {
      hasSelfMapping: true,
      tourId,
      optionId,
    });
    await this.$store.dispatch("pricing/setConnectivityType", "availability_and_price");

    this.showChangingFromPricesOverApiModal = false;
  }

  onContinueManuallyClick() {
    const { tourId, optionId } = this;

    this.resetConnectivityType();
    this.resetPricingType();

    this.$store.dispatch("pricing/setHasSelfMapping", {
      hasSelfMapping: false,
      tourId,
      optionId,
    });
    this.$store.dispatch("pricing/setSelfMappingValidity");
    this.$store.dispatch("pricing/setSelfMappingErrors", []);
  }
}
</script>

<style lang="scss" scoped>
@import "~@Assets/styles/variables";

.self-mapping {
  &__alert {
    margin-top: $base-unit * 2;
  }

  .radio-buttons-container.self-mapping-selection {
    margin: $base-unit * 2 0;
  }

  .connectivity-type {
    margin-top: $base-unit * 4;
  }

  .self-mapping-alert {
    max-width: 500px;
  }

  .self-mapping-form-item {
    margin: $base-unit * 2 0;

    select {
      min-width: 360px;
    }
  }

  .save-button-container {
    justify-content: left;

    .btn-ghost {
      margin-left: $base-unit;
    }
  }

  .instruction {
    margin-left: $base-unit * 2;
    list-style-type: disc !important;
    padding: 0 !important;
  }

  .instructions_header {
    margin-bottom: $base-unit * 3;
  }

  .instructions_footer {
    margin-top: $base-unit * 3;
  }

  .availability-instructions {
    margin-top: $base-unit * 3;
    margin-bottom: $base-unit * 3;
  }
}
</style>
