<template>
  <form
    class="modal-form"
    @submit.prevent="submit"
  >

    <!-- Input Area -->
    <div class="modal-send-receive-input-area">

      <div class="block text-left mt-[25px]">
        <CurrencyField
          id="amountSupply"
          :currency-options="modelValue.currentBalance"
          :error-msg="modelValue.amountErrorMsg"
          :is-error="modelValue.amountErrorMsg !== ''"
          :option="modelValue.selectedCurrency"
          :value="modelValue.amount"
          :label="$t('message.amount')"
          :balance="formatCurrentBalance(modelValue.selectedCurrency)"
          :total="modelValue.selectedCurrency?.balance"
          :set-input-value="setAmount"
          :disabled-currency-picker="true"
          name="amountSupply"
          @input="handleAmountChange($event)"
          @update-currency="(event) => (modelValue.selectedCurrency = event)"
        />
      </div>
    </div>
    <!-- Actions -->
    <div class="modal-send-receive-actions">
      <button
        class="btn btn-primary btn-large-primary text-center min-h-[44px]"
        :class="{ 'js-loading': props.modelValue.loading }"
        :disabled="!props.modelValue.supply"
      >
        {{ props.modelValue.loading ? '' : props.modelValue.supply ? $t("message.supply") : $t("message.supply-limit-reached") }}
      </button>
    </div>
  </form>
</template>

<script lang="ts" setup>
import type { SupplyFormComponentProps } from "@/types/component/SupplyFormComponentProps";
import { type PropType } from "vue";
import type { AssetBalance } from "@/stores/wallet/state";

import CurrencyField from "@/components/CurrencyField.vue";
import { CurrencyUtils } from "@nolus/nolusjs";
import { useWalletStore } from "@/stores/wallet";
import { Dec } from "@keplr-wallet/unit";

const props = defineProps({
  modelValue: {
    type: Object as PropType<SupplyFormComponentProps>,
    required: true,
  },
});

const wallet = useWalletStore();

const submit = () => {
  if (props.modelValue.supply) {
    props.modelValue.onNextClick();
  }
}

const handleAmountChange = (value: string) => {
  props.modelValue.amount = value;
};

const formatCurrentBalance = (selectedCurrency: AssetBalance) => {
  if (selectedCurrency?.balance?.denom && selectedCurrency?.balance?.amount) {
    const asset = wallet.getCurrencyInfo(
      props.modelValue.selectedCurrency.balance.denom
    );
    return CurrencyUtils.convertMinimalDenomToDenom(
      selectedCurrency.balance.amount.toString(),
      selectedCurrency.balance.denom,
      asset.shortName,
      asset.coinDecimals
    ).toString();
  }
};

const setAmount = (p: number) => {
  const asset = wallet.getCurrencyInfo(
    props.modelValue.selectedCurrency.balance.denom
  );
  const percent = new Dec(p).quo(new Dec(100));
  const amount = CurrencyUtils.convertMinimalDenomToDenom(props.modelValue.selectedCurrency.balance.amount, asset.coinMinimalDenom, asset.coinDenom, asset.coinDecimals).toDec();
  const value = amount.mul(percent);
  props.modelValue.amount = value.toString(asset.coinDecimals);
};

defineEmits(["update:modelValue.selectedCurrency"]);
</script>
