<template>
  <ConfirmExternalComponent
    v-if="showConfirmScreen"
    :network-type="NetworkTypes.cosmos"
    :selectedCurrency="selectedCurrency!"
    :receiverAddress="walletStore.wallet?.address as string"
    :password="password"
    :amount="amount"
    :memo="memo"
    :txType="$t(`message.${TxType.SEND}`) + ':'"
    :txHash="txHash"
    :step="step"
    :fee="fee"
    :networkCurrencies="networkCurrenciesObject"
    :networkKey="selectedNetwork.key"
    :networkSymbol="selectedNetwork.symbol"
    :onSendClick="onSendClick"
    :onBackClick="onConfirmBackClick"
    :onOkClick="onClickOkBtn"
    @passwordUpdate="(value) => (password = value)"
  />
  <template v-else>
    <div
      class="modal-send-receive-input-area overflow-auto custom-scroll"
      v-if="selectedNetwork.native"
    >
      <div class="block text-left">
        <div class="block mt-[25px]">
          <Picker
            :default-option="networks[0]"
            :options="networks"
            :label="$t('message.network')"
            @update-selected="onUpdateNetwork"
          />
        </div>

        <div class="block mt-[18px]">
          <p class="text-14 nls-font-500 text-primary m-0 mb-[6px]">
            {{ $t("message.address") }}
          </p>
          <p class="text-14 text-primary nls-font-700 m-0 break-all">
            {{
              WalletUtils.isAuth()
                ? walletStore.wallet?.address
                : $t("message.connect-wallet-label")
            }}
          </p>
          <div class="flex items-center justify-start mt-2">
            <button
              class="btn btn-secondary btn-medium-secondary btn-icon mr-2 flex"
              @click="
                modelValue?.onCopyClick(wallet);
                onCopy();
              "
            >
              <DocumentDuplicateIcon class="icon w-4 h-4" />
              {{ copyText }}
            </button>
          </div>
        </div>

        <div class="flex justify-between w-full text-light-blue text-[14px] mt-4">
          <p>{{ $t("message.estimate-time") }}:</p>
          <p>~{{ selectedNetwork.estimation }} {{ $t("message.sec") }}</p>
        </div>
      </div>
    </div>
    <template v-else>
      <form @submit.prevent="receive" class="modal-form overflow-auto">
        <!-- Input Area -->
        <div class="modal-send-receive-input-area background">
          <div class="block text-left">
            <div class="block mt-[20px]">
              <Picker
                :default-option="selectedNetwork"
                :options="networks"
                :label="$t('message.network')"
                :value="selectedNetwork"
                @update-selected="onUpdateNetwork"
              />
            </div>

            <div class="block mt-[20px]">
              <CurrencyField
                id="amount"
                :currency-options="networkCurrencies"
                :disabled-currency-picker="disablePicker || disablePickerDialog"
                :is-loading-picker="disablePicker"
                :error-msg="amountErrorMsg"
                :is-error="amountErrorMsg !== ''"
                :option="selectedCurrency"
                :value="amount"
                :name="$t('message.amount')"
                :label="$t('message.amount-receive')"
                :total="total"
                :balance="formatCurrentBalance(selectedCurrency)"
                @update-currency="(event: AssetBalance) => (selectedCurrency = event)"
                @input="handleAmountChange($event)"
                :set-input-value="setAmountValue"
              />
            </div>

            <div>
              <p class="text-14 nls-font-500 text-primary m-0 mb-[6px] mt-4">
                {{ $t("message.recipient") }}
              </p>
              <p class="text-14 text-primary nls-font-700 m-0 break-all">
                {{
                  WalletUtils.isAuth()
                    ? walletStore.wallet?.address
                    : $t("message.connect-wallet-label")
                }}
              </p>
            </div>
          </div>
        </div>
        <!-- Actions -->
        <div class="modal-send-receive-actions background flex-col">
          <button class="btn btn-primary btn-large-primary" :class="{ 'js-loading': isLoading }">
            {{ $t("message.receive") }}
          </button>
          <div class="flex justify-between w-full text-light-blue text-[14px] my-2">
            <p>{{ $t("message.estimate-time") }}:</p>
            <p>~{{ selectedNetwork.estimation }} {{ $t("message.sec") }}</p>
          </div>
        </div>
      </form>
    </template>
  </template>
</template>

<script setup lang="ts">
import Picker from "@/components/Picker.vue";
import CurrencyField from "@/components/CurrencyField.vue";
import ConfirmExternalComponent from "@/components/modals/templates/ConfirmExternalComponent.vue";

import type { AssetBalance } from "@/stores/wallet/state";
import type { ExternalCurrencyType } from "@/types/CurreciesType";
import { CONFIRM_STEP, TxType, type Network, NetworkTypes } from "@/types";

import { onUnmounted, ref, type PropType, inject, watch, computed } from "vue";
import { DocumentDuplicateIcon } from "@heroicons/vue/24/solid";
import {
CurrencyDemapping,
  CurrencyMapping,
  ErrorCodes,
  IGNORE_TRANSFER_ASSETS,
  LPN_NETWORK,
  NATIVE_NETWORK,
  SOURCE_PORTS
} from "@/config/env";
import { useI18n } from "vue-i18n";
import { AssetUtils, EnvNetworkUtils, WalletUtils } from "@/utils";
import { NETWORKS_DATA, SUPPORTED_NETWORKS_DATA } from "@/networks/config";
import { Wallet, BaseWallet } from "@/networks";
import { coin, type Coin } from "@cosmjs/amino";
import { Decimal } from "@cosmjs/math";
import { externalWalletOperation, externalWallet } from "../utils";
import { CurrencyUtils } from "@nolus/nolusjs";
import { WalletActionTypes, useWalletStore } from "@/stores/wallet";
import { Dec, Coin as KeplrCoin } from "@keplr-wallet/unit";
import { useApplicationStore } from "@/stores/application";
import { onMounted } from "vue";
import { AssetUtils as NolusAssetUtils } from "@nolus/nolusjs/build/utils/AssetUtils";
import { Networks, Protocols } from "@nolus/nolusjs/build/types/Networks";

export interface ReceiveComponentProps {
  currentBalance: AssetBalance[];
  selectedCurrency: AssetBalance;
  amount: string;
  dialogSelectedCurrency: string;
  onScanClick: () => void;
  onCopyClick: (wallet?: string) => void;
}

let timeOut: NodeJS.Timeout;
let client: Wallet;
const props = defineProps({
  modelValue: {
    type: Object as PropType<ReceiveComponentProps>
  }
});

const walletStore = useWalletStore();
const app = useApplicationStore();

const networks = computed(() => {
  const n: string[] = [];
  if ((props.modelValue?.dialogSelectedCurrency.length as number) > 0) {
    const [ckey, protocol]: string[] = props.modelValue!.dialogSelectedCurrency.split("@");

    let lpn = app.lpn?.find((item) => {
      return item.key == props.modelValue!.dialogSelectedCurrency;
    });

    n.push(NATIVE_NETWORK.key);

    if (ckey == NATIVE_NETWORK.symbol) {
      n.push(Protocols.neutron);
    }

    if (lpn) {
      const [key, protocol] = lpn.key!.split("@");
      n.push(protocol);

      for (const ntw of LPN_NETWORK) {
        n.push(ntw);
      }
    } else {
      for (const key in app.networks ?? {}) {
        if (app.networks?.[key][ckey]) {
          n.push(key);
        }
      }
    }

    return NETWORKS_DATA[EnvNetworkUtils.getStoredNetworkName()].list.filter((item) =>
      n.includes(item.key)
    );
  }
  return NETWORKS_DATA[EnvNetworkUtils.getStoredNetworkName()].list;
});

const i18n = useI18n();
const copyText = ref(i18n.t("message.copy"));
const selectedNetwork = ref(networks.value[0]);

const networkCurrencies = ref<AssetBalance[]>([]);
const selectedCurrency = ref<AssetBalance>(walletStore.balances[0]);
const amount = ref("");
const amountErrorMsg = ref("");
const disablePicker = ref(false);
const disablePickerDialog = ref(false);

const showConfirmScreen = ref(false);
const step = ref(CONFIRM_STEP.CONFIRM);
const password = ref("");
const memo = ref("");
const txHash = ref("");
const fee = ref<Coin>();
const isLoading = ref(false);
const closeModal = inject("onModalClose", () => () => {});
const networkCurrenciesObject = ref();
let wallet = ref(walletStore.wallet?.address);

onMounted(() => {
  if ((props.modelValue?.dialogSelectedCurrency.length as number) > 0) {
    disablePickerDialog.value = true;
    const [_ckey, network]: string[] = props.modelValue!.dialogSelectedCurrency.split("@");
    onUpdateNetwork(SUPPORTED_NETWORKS_DATA[network ?? Protocols.osmosis] as Network);
  }
});

onUnmounted(() => {
  clearTimeout(timeOut);
  if (client) {
    client.destroy();
  }
});

watch(
  () => [selectedCurrency.value, amount.value],
  () => {
    if (amount.value.length > 0) {
      validateAmount();
    }
  }
);

const onUpdateNetwork = async (event: Network) => {
  selectedNetwork.value = event;
  networkCurrencies.value = [];

  if (!event.native) {
    if (client) {
      client.destroy();
    }

    disablePicker.value = true;

    const network = NETWORKS_DATA[EnvNetworkUtils.getStoredNetworkName()];
    const assets = network.supportedNetworks[event.key].currencies();
    const currenciesPromise = [];
    const filteredAssets: { [key: string]: ExternalCurrencyType } = {};
    const networkData = network?.supportedNetworks[selectedNetwork.value.key];

    for (const key in assets) {
      if (LPN_NETWORK.includes(selectedNetwork.value.key)) {
        let lpn = app.lpn?.find((item) => {
          return item.ticker == key;
        });
        if (lpn) {
          for (const lpn of app.lpn ?? []) {
            filteredAssets[lpn.key as string] = {
              native: true,
              decimal_digits: lpn.decimal_digits,
              ibcData: assets[key].ibcData,
              key: lpn.key,
              name: lpn.name,
              shortName: lpn.shortName,
              symbol: lpn.symbol,
              ticker: lpn.ticker
            };
          }
          continue;
        }
      }
      if (!IGNORE_TRANSFER_ASSETS.includes(key)) {
        filteredAssets[key] = assets[key];
      }
    }

    client = await WalletUtils.getWallet(event.key);

    const baseWallet = (await externalWallet(client, networkData, password.value)) as BaseWallet;
    wallet.value = baseWallet?.address as string;
    networkCurrenciesObject.value = filteredAssets;

    for (const key in filteredAssets) {
      const fn = async () => {
        let k = filteredAssets[key].key!;
        let [ckey, protocol = Protocols.osmosis]: string[] = k.split("@");

        if (LPN_NETWORK.includes(selectedNetwork.value.key)) {
          let lpn = app.lpn?.find((item) => {
            return item.key == k;
          });
          if (lpn) {
            ckey = filteredAssets[key].ticker;
          }
        }

        const ibc_route = NolusAssetUtils.makeIBCMinimalDenom(
          ckey,
          app.networksData!,
          networkData.key as Networks,
          protocol as Protocols
        );

        const balance = WalletUtils.isAuth()
          ? await client.getBalance(wallet.value as string, ibc_route)
          : coin(0, ibc_route);

        let shortName = filteredAssets[key].shortName;
        let [ticker] = filteredAssets[key].key?.split("@") as string[];

        if (CurrencyMapping[key]) {
          shortName = CurrencyMapping[key].name ?? shortName;
          ticker = key;
        }

        if (CurrencyDemapping[ckey]) {
          shortName = CurrencyDemapping[ckey].name ?? shortName;
        }

        switch (selectedNetwork.value.key) {
          case Protocols.neutron: {
            protocol = Protocols.neutron;
            break;
          }
        }

        const icon = app.assetIcons?.[`${ticker}@${protocol}`] as string;

        return {
          balance,
          shortName: shortName,
          ticker: k,
          name: shortName,
          icon: icon,
          decimals: Number(filteredAssets[key].decimal_digits),
          symbol: filteredAssets[key].symbol,
          native: filteredAssets[key].native
        };
      };

      currenciesPromise.push(fn());
    }

    const items = await Promise.all(currenciesPromise);
    if ((props.modelValue?.dialogSelectedCurrency.length as number) > 0) {
      const [ckey]: string[] = props.modelValue!.dialogSelectedCurrency.split("@");
      const c = items.find((e) => e.ticker == ckey || e.ticker == props.modelValue!.dialogSelectedCurrency)!;
      selectedCurrency.value = c;
    } else {
      selectedCurrency.value = items?.[0];
    }
    networkCurrencies.value = items;
    disablePicker.value = false;
  } else {
    selectedCurrency.value = walletStore.balances[0];
    wallet.value = walletStore.wallet?.address;
  }
};

const onCopy = () => {
  copyText.value = i18n.t("message.copied");
  if (timeOut) {
    clearTimeout(timeOut);
  }
  timeOut = setTimeout(() => {
    copyText.value = i18n.t("message.copy");
  }, 2000);
};

const receive = () => {
  validateInputs();
};

const handleAmountChange = (event: string) => {
  amount.value = event;
};

const validateInputs = async () => {
  try {
    isLoading.value = true;
    const isValid = await validateAmount();
    if (isValid) {
      const network =
        NETWORKS_DATA[EnvNetworkUtils.getStoredNetworkName()]?.supportedNetworks[
          selectedNetwork.value.key
        ];

      fee.value = coin(network.fees.transfer_amount, selectedCurrency.value.balance.denom);
      showConfirmScreen.value = true;
    }
  } catch (error) {
    step.value = CONFIRM_STEP.ERROR;
    showConfirmScreen.value = true;
  } finally {
    isLoading.value = false;
  }
};

const validateAmount = async () => {
  amountErrorMsg.value = "";

  if (!WalletUtils.isAuth()) {
    return false;
  }

  if (!amount.value) {
    amountErrorMsg.value = i18n.t("message.invalid-amount");
    return false;
  }

  const prefix =
    NETWORKS_DATA[EnvNetworkUtils.getStoredNetworkName()]?.supportedNetworks[
      selectedNetwork.value.key
    ]?.prefix;
  const decimals = selectedCurrency.value?.decimals;

  if (prefix && decimals) {
    try {
      const balance = await client.getBalance(
        wallet.value as string,
        selectedCurrency.value.balance.denom
      );
      const walletBalance = Decimal.fromAtomics(balance.amount, decimals);
      const transferAmount = Decimal.fromUserInput(amount.value, decimals);
      const isGreaterThanWalletBalance = transferAmount.isGreaterThan(walletBalance);

      if (isGreaterThanWalletBalance) {
        amountErrorMsg.value = i18n.t("message.invalid-balance-big");
        return false;
      }
    } catch (e) {
      console.log(e);
    }
  } else {
    amountErrorMsg.value = i18n.t("message.unexpected-error");
    return false;
  }

  return true;
};

const onSendClick = async () => {
  try {
    const networkData =
      NETWORKS_DATA[EnvNetworkUtils.getStoredNetworkName()]?.supportedNetworks[
        selectedNetwork.value.key
      ];
    await externalWalletOperation(ibcTransfer, client, networkData, password.value);
  } catch (error: Error | any) {
    step.value = CONFIRM_STEP.ERROR;
  }
};

const getSourceChannel = () => {
  const networkInfo =
    SUPPORTED_NETWORKS_DATA[selectedNetwork.value.key as keyof typeof SUPPORTED_NETWORKS_DATA];

  if (networkInfo.forward) {
    const [_ckey, protocol = Protocols.osmosis]: string[] = selectedCurrency.value.ticker!.split('@');

    return AssetUtils.getChannelDataByProtocol(
      app.networksData?.networks?.channels!,
      protocol,
      selectedNetwork.value.key
    )!.b.ch;
  } else {
    return AssetUtils.getSourceChannel(
      app.networksData?.networks?.channels!,
      selectedNetwork.value.key,
      NATIVE_NETWORK.key,
      selectedNetwork.value.key
    );
  }
};

const ibcTransfer = async (baseWallet: BaseWallet) => {
  try {
    step.value = CONFIRM_STEP.PENDING;

    const networkData =
      NETWORKS_DATA[EnvNetworkUtils.getStoredNetworkName()]?.supportedNetworks[
        selectedNetwork.value.key
      ];
    const denom = selectedCurrency.value.balance.denom;

    const minimalDenom = CurrencyUtils.convertDenomToMinimalDenom(
      amount.value,
      denom,
      selectedCurrency.value?.decimals!
    );

    const funds: Coin = {
      amount: minimalDenom.amount.toString(),
      denom
    };

    const sourceChannel = getSourceChannel();
    const networkInfo =
      SUPPORTED_NETWORKS_DATA[selectedNetwork.value.key as keyof typeof SUPPORTED_NETWORKS_DATA];

    const rawTx: {
      toAddress: string;
      amount: Coin;
      sourcePort: string;
      sourceChannel: string;
      gasMuplttiplier: number;
      gasPrice: string;
      timeOut: number;
      memo?: string;
    } = {
      toAddress: walletStore.wallet?.address as string,
      amount: funds,
      sourcePort: SOURCE_PORTS.TRANSFER,
      sourceChannel: sourceChannel as string,
      gasMuplttiplier: networkData.gasMuplttiplier,
      gasPrice: networkData.gasPrice,
      timeOut: networkData.ibcTransferTimeout
    };

    if (networkInfo.forward) {
      const [_ckey, protocol = Protocols.osmosis]: string[] = selectedCurrency.value.ticker!.split('@');
      const proxyAddress = wallet.value as string;
      const channel = AssetUtils.getSourceChannel(
        app.networksData?.networks?.channels!,
        NATIVE_NETWORK.key,
        protocol,
        protocol
      );

      rawTx.toAddress = proxyAddress;

      rawTx.memo = JSON.stringify({
        forward: {
          receiver: walletStore.wallet!.address,
          port: SOURCE_PORTS.TRANSFER,
          channel: channel
        }
      });
    }

    const {
      txHash: txHashData,
      txBytes,
      usedFee
    } = await baseWallet.simulateSendIbcTokensTx(rawTx);
    txHash.value = txHashData;

    if (usedFee?.amount?.[0]) {
      fee.value = usedFee.amount[0];
    }

    const tx = await baseWallet.broadcastTx(txBytes as Uint8Array);
    const isSuccessful = tx?.code === 0;
    step.value = isSuccessful ? CONFIRM_STEP.SUCCESS : CONFIRM_STEP.ERROR;
    baseWallet.disconnect();

    setTimeout(async () => {
      await walletStore[WalletActionTypes.UPDATE_BALANCES]();
    }, 10000);
  } catch (error: Error | any) {
    switch (error.code) {
      case ErrorCodes.GasError: {
        step.value = CONFIRM_STEP.GasErrorExternal;
        break;
      }
      default: {
        step.value = CONFIRM_STEP.ERROR;
        break;
      }
    }

    if (error.message.includes("You might want to check later")) {
      step.value = CONFIRM_STEP.SUCCESS;
      return;
    }

    if (error.message.includes("Length must be a multiple of 4")) {
      step.value = CONFIRM_STEP.SUCCESS;
      return;
    }
  }
};

const onConfirmBackClick = () => {
  showConfirmScreen.value = false;
};

const onClickOkBtn = () => {
  closeModal();
};

const formatCurrentBalance = (selectedCurrency: AssetBalance | undefined) => {
  if (selectedCurrency?.balance?.denom && selectedCurrency?.balance?.amount) {
    if (selectedNetwork.value.native) {
      const asset = walletStore.getCurrencyInfo(selectedCurrency.balance.denom);
      return CurrencyUtils.convertMinimalDenomToDenom(
        selectedCurrency.balance.amount.toString(),
        selectedCurrency.balance.denom,
        asset.coinDenom,
        asset.coinDecimals
      ).toString();
    } else {
      if (selectedCurrency.decimals != null && selectedCurrency.name != null) {
        return CurrencyUtils.convertMinimalDenomToDenom(
          selectedCurrency.balance.amount.toString(),
          selectedCurrency.balance.denom,
          selectedCurrency.name as string,
          selectedCurrency.decimals as number
        ).toString();
      }
    }
  }
};

const setAmountValue = (p: number) => {
  const asset = AssetUtils.getAssetInfo(selectedCurrency.value.ticker as string);
  const percent = new Dec(p).quo(new Dec(100));
  const data = CurrencyUtils.convertMinimalDenomToDenom(
    selectedCurrency.value.balance.amount,
    asset.coinMinimalDenom,
    asset.coinDenom,
    asset.coinDecimals
  ).toDec();
  const value = data.mul(percent);
  amount.value = value.toString(asset.coinDecimals);
};

const total = computed(() => {
  if (selectedCurrency.value) {
    return new KeplrCoin(
      selectedCurrency.value.balance.denom,
      selectedCurrency.value.balance.amount
    );
  }
  return undefined;
});
</script>
