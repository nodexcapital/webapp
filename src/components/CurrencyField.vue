<template>
  <div class="block currency-field-container">
    <div class="flex justify-between items-center">
      <label
        :for="id"
        class="flex text-14 nls-font-500 data-text"
      >
        {{ label }}
        <TooltipComponent
          v-if="tooltip.length > 0"
          :content="tooltip"
        />
      </label>
      <div
        v-if="balance"
        class="balance select-none cursor-pointer"
        @click="setBalance"
      >
        {{ $t('message.balance') }} {{ balance }}
      </div>
    </div>

    <div
      class="currency-field p-2.5 currency-field p-3.5"
      :class="{ error: isError }"
    >
      <div class="flex items-center">
        <div class="inline-block">
          <CurrencyPicker
            :currency-option="option"
            :disabled="disabledCurrencyPicker"
            :options="currencyOptions"
            @update-currency="onUpdateCurrency"
            :isLoading="isLoadingPicker"
            type="small"
          />
        </div>
        <div class="inline-block flex-1">
          <input
            :id="id"
            :disabled="disabledInputField"
            :name="name"
            v-model="numberValue"
            autocomplete="off"
            class="nls-font-700 text-18 text-primary background text-right"
            @keydown="inputValue"
            @keypress.space.prevent
            @paste="onPaste"
            @keyup="setValue"
            :placeholder="placeholder"
          />
          <span class="block text-14 nls-font-400 text-light-blue text-right">
            {{ calculateInputBalance() }}
          </span>
        </div>
      </div>
    </div>

    <!-- <div class="repayment items-start justify-between">
      <span class="msg error">
        {{ errorMsg }}
      </span>
      <div class="min-w-[156px]">
        <button
          v-for="value in INPUT_VALUES"
          type="button"
          @click="setInputValue(value)"
          :key="value"
        >
          {{ value }}%
        </button>
      </div>
    </div> -->


  </div>
</template>

<script setup lang="ts">
import type { AssetBalance } from "@/stores/wallet/state";
import { onMounted, ref, watch, type PropType } from "vue";

import CurrencyPicker from "@/components/CurrencyPicker.vue";
import TooltipComponent from "./TooltipComponent.vue";

import { Coin, Dec, Int } from "@keplr-wallet/unit";
import { CurrencyUtils } from "@nolus/nolusjs";
import { useOracleStore } from "@/stores/oracle";
import { useWalletStore } from "@/stores/wallet";
import type { ExternalCurrencyType } from "@/types/CurreciesType";

const emit = defineEmits(["update-currency", "update:modelValue", "input"]);
const oracle = useOracleStore();
const wallet = useWalletStore();

const dot = ".";
const minus = "-";
const comma = ",";
const allowed = [
  "Delete",
  "Backspace",
  "ArrowLeft",
  "ArrowRight",
  "-",
  ".",
  "Enter",
  "Tab",
  "Control",
  "End",
  "Home"
];
const numbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

const props = defineProps({
  name: {
    type: String,
  },
  value: {
    type: String,
  },
  currencyOptions: {
    type: Array as PropType<AssetBalance[]>,
  },
  tooltip: {
    type: String,
    default: ''
  },
  option: {
    type: Object as PropType<AssetBalance>,
  },
  id: {
    type: String,
  },
  label: {
    type: String,
  },
  disabledInputField: {
    type: Boolean,
  },
  disabledCurrencyPicker: {
    type: Boolean,
  },
  isLoadingPicker: {
    type: Boolean,
    default: false
  },
  isError: {
    type: Boolean,
    default: false,
  },
  errorMsg: {
    type: String,
    default: "",
  },
  placeholder: {
    type: String,
    default: "0"
  },
  balance: {
    type: String
  },
  total: {
    type: Object,
  },
  setInputValue: {
    type: Function,
    required: true
  },
  price: {
    type: Number
  },
  positive: {
    type: Boolean,
    default: false,
  },
});

const numberValue = ref(props.value);
let numberRealValue = Number(props.value);

onMounted(() => {
  setValue();
});

watch(
  () => props.value,
  () => {
    numberValue.value = props.value;
    numberRealValue = Number(props.value);
    setValue();
  }
);

const onUpdateCurrency = (value: AssetBalance) => {
  emit("update-currency", value);
};

const calculateInputBalance = () => {

  if (props.price) {
    const coin = CurrencyUtils.convertDenomToMinimalDenom(
      numberRealValue.toString(),
      props.option?.balance.denom as string,
      props.option?.decimals as number
    );
    return CurrencyUtils.calculateBalance(props.price.toString(), coin, props.option?.decimals as number);
  }

  const prices = oracle.prices;

  if (!numberRealValue || !props.option || !prices) {
    return "$0";
  }

  const ticker = props.option.ticker;

  let coinDecimals = null;
  let coinMinimalDenom = null;
  let symbol = null;

  if (ticker) {
    const { decimals, symbol: currencySymbol } = props.option;
    coinDecimals = decimals as number;
    coinMinimalDenom = props.option.balance.denom
    symbol = currencySymbol as string;
  } else {
    const denom = props.option.balance.denom;
    const { coinDecimals: decimals, coinMinimalDenom: minimalDenom } = wallet.getCurrencyInfo(denom);
    symbol = wallet.currencies[denom].symbol;
    coinDecimals = decimals;
    coinMinimalDenom = minimalDenom;
  }

  const { amount } = CurrencyUtils.convertDenomToMinimalDenom(
    numberRealValue.toString(),
    coinMinimalDenom as string,
    coinDecimals
  );

  const coin = new Coin(coinMinimalDenom as string, new Int(String(amount)));
  const tokenPrice = prices[coinMinimalDenom]?.amount || "0";

  return CurrencyUtils.calculateBalance(tokenPrice, coin, coinDecimals);
};

const inputValue = (event: KeyboardEvent) => {
  const charCode = event.key;
  const value = numberValue.value ?? "";

  if (event.ctrlKey || event.metaKey) {
    return true;
  }

  if (props.positive) {
    if (event.key == minus) {
      event.preventDefault();
      return false;
    }
  }

  if (charCode == minus && value.includes(minus)) {
    event.preventDefault();
    return false;
  }

  if (charCode == dot && value?.includes(dot)) {
    event.preventDefault();
    return false;
  }

  if (allowed.includes(charCode)) {
    return true;
  }

  const num = Number(charCode);
  if (numbers.includes(num)) {
    return true;
  }

  event.preventDefault();
  return false;
};

const onPaste = (event: ClipboardEvent) => {
  const pastedText = event.clipboardData?.getData("text");
  const num = Number(pastedText);
  if (isNaN(num)) {
    event.preventDefault();
  }
};

const setValue = () => {
  let value = removeComma(numberValue.value ?? "");
  let numValue = Number(value);
  numberValue.value = commify(value.toString());
  if (isNaN(numValue)) {
    return false;
  }
  numberRealValue = Number(value);
  emit("input", value);
  emit("update:modelValue", value);
};

const setBalance = () => {
  if (props.total) {
    const currency: ExternalCurrencyType | any = props.option?.ticker ? wallet.getCurrencyByTicker(props.option?.ticker) : wallet.getCurrencyInfo(props.total.denom);
    const decimals = Number(currency.decimal_digits ?? currency.coinDecimals)
    const value = new Dec(props.total.amount, decimals);
    emit("input", value.toString(decimals));
    emit("update:modelValue", value.toString(decimals));
  }
}

const commify = (n: string) => {
  const parts = n.split(".");
  const numberPart = parts[0];
  const decimalPart = parts[1];
  const hasDot = n.includes(dot);
  const thousands = /\B(?=(\d{3})+(?!\d))/g;

  return (
    numberPart.replace(thousands, comma) + (hasDot ? `.${decimalPart}` : "")
  );
};

const removeComma = (n: string) => {
  const re = new RegExp(comma, "g");
  return n.replace(re, "");
};
</script>
