<script setup>
import { ref, computed, onMounted } from "vue";

const props = defineProps({
  cryptosList: {
    type: Array,
    required: true,
    default: () => [],
  },
});

const selectedCrypto = ref(
  props.cryptosList.length > 0 ? props.cryptosList[0].id : "",
);

const currentCryptoData = computed(() => {
  return props.cryptosList.find((c) => c.id === selectedCrypto.value);
});

const notifySelection = () => {
  if (!selectedCrypto.value) return;

  window.__selectedCrypto = selectedCrypto.value;
  const event = new CustomEvent("crypto-changed", {
    detail: { coin: selectedCrypto.value },
  });
  window.dispatchEvent(event);
};

onMounted(() => {
  notifySelection();
});
</script>

<template>
  <div class="space-y-4">
    <div>
      <label
        for="crypto-select"
        class="block text-sm font-medium text-slate-700 mb-1"
      >
        Selecciona una criptomoneda:
      </label>
      <select
        id="crypto-select"
        v-model="selectedCrypto"
        @change="notifySelection"
        class="w-full bg-slate-50 border border-slate-200 text-slate-800 rounded-lg p-2.5 outline-none focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 transition-all text-sm"
      >
        <option value="" disabled>Selecciona una moneda</option>

        <option
          v-for="crypto in cryptosList"
          :key="crypto.id"
          :value="crypto.id"
        >
          {{ crypto.name ?? crypto.id }} ({{ crypto.symbol?.toUpperCase() ?? "N/A" }})
        </option>
      </select>
    </div>

    <div
      v-if="currentCryptoData"
      class="mt-4 p-4 bg-slate-50 border border-slate-100 rounded-lg text-sm"
    >
      <div class="flex items-center space-x-2">
        <img
          v-if="currentCryptoData.image"
          :src="currentCryptoData.image"
          :alt="currentCryptoData.name"
          class="w-6 h-6 rounded-full"
        />
        <span class="font-bold text-slate-800">{{
          currentCryptoData.name
        }}</span>
      </div>
      <p class="text-slate-500 mt-2">
        Precio Base (USD):
        <span class="text-slate-800 font-semibold"
          >${{ currentCryptoData.current_price?.toLocaleString() ?? "No disponible" }}</span
        >
      </p>
      <p class="text-slate-500">
        Cap. de Mercado:
        <span class="text-slate-800 font-semibold"
          >${{ currentCryptoData.market_cap?.toLocaleString() ?? "No disponible" }}</span
        >
      </p>
    </div>
  </div>
</template>
