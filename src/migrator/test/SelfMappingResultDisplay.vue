<template>
  <div class="self-mapping-result-display">
    <div v-if="showSuccessBanner && hasSupportedSystem && selfMapping.isValid">
      <GygAlert modifier="success" :closable="false">
        {{ $t("pAvailability_cSelfMapping_SuccessNotification", [reservationSystemName]) }}
      </GygAlert>

      <div class="self-mapping-result-section">
        <p class="section-title">{{ $t("pAvailability_cSelfMapping_Account") }}</p>
        <p v-html-safe="connectivitySystem"></p>
        <p v-html-safe="supplierInfo"></p>
      </div>

      <div class="self-mapping-result-section">
        <p class="section-title">{{ $t("pAvailability_cSelfMapping_Product") }}</p>
        <p v-html-safe="externalOptionInfo"></p>
        <p v-html-safe="optionInfo"></p>
      </div>

      <div class="self-mapping-result-section">
        <p class="section-title">{{ $t("pAvailability_cSelfMapping_ConnectivityType") }}</p>
        <p>{{ $t("pAvailability_cSelfMapping_Availability") }}</p>
        <p v-if="connectivityType === 'availability_and_price'">
          {{ $t("pAvailability_cSelfMapping_Pricing") }}
        </p>
      </div>
    </div>

    <div v-if="selfMapping.errors.length">
      <GygAlert
        v-for="(error, index) in selfMapping.errors"
        :key="index"
        modifier="error"
        :closable="false"
      >
        <span v-if="error" v-html-safe="error"></span>
        <span v-else v-html-safe="$t('pGlobal_cErrorTemplates_GeneralError')" />
      </GygAlert>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";
import { GygAlert } from "@getyourguide/design-system";
import { TranslateResult } from "vue-i18n";

@Component({
  components: {
    GygAlert,
  },
})
export default class SelfMappingResultDisplay extends Vue {
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
  @Prop({ type: Number, required: true }) readonly supplierId!: number;
  @Prop({ type: String, required: true }) readonly companyName!: string;
  @Prop({ type: [Number, String], required: true }) readonly optionId!: number | string;
  @Prop({ type: String, required: true }) readonly optionTitle!: string;
  @Prop({ type: String, required: true }) readonly connectivityType!: string | null;
  @Prop({ type: Boolean, required: true }) readonly hasSupportedSystem!: boolean;
  @Prop({ type: Boolean, required: true }) readonly showSuccessBanner!: boolean;

  get reservationSystemName(): string {
    let reservationSystem = "";
    this.selfMapping.availableReservationSystems.value.forEach(({ text, value }) => {
      if (this.selfMapping.reservationSystem.value === value) {
        reservationSystem = text;
      }
    });

    return reservationSystem;
  }

  get connectivitySystem(): TranslateResult {
    return this.$t("pAvailability_cSelfMapping_ConnectivitySystem", [this.reservationSystemName]);
  }

  get supplierInfo(): TranslateResult {
    return this.$t("pAvailability_cSelfMapping_SupplierId", [this.supplierId, this.companyName]);
  }

  get externalOptionInfo(): TranslateResult {
    return this.$t("pAvailability_cSelfMapping_ExternalOptionId", [
      this.reservationSystemName,
      this.selfMapping.externalProductId.value,
    ]);
  }

  get optionInfo(): TranslateResult {
    return this.$t("pAvailability_cSelfMapping_OptionId", [this.optionId, this.optionTitle]);
  }
}
</script>

<style lang="scss">
@import "~@Assets/styles/variables";

.self-mapping-result-display {
  .self-mapping-result-section {
    margin-bottom: $base-unit * 2;

    p {
      margin-bottom: $half-base-unit;
    }
  }
}
</style>
