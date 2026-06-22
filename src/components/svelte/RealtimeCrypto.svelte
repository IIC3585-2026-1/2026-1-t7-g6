<script lang="ts">
  import { onDestroy, onMount } from "svelte";

  export let initialCoin = "bitcoin";

  type PricePoint = {
    price: number;
    timestamp: number;
  };

  type CryptoWindow = Window & {
    __selectedCrypto?: string;
  };

  type PriceResponse = Record<
    string,
    { usd?: number; last_updated_at?: number }
  >;

  const POLLING_INTERVAL = 20_000;
  const MAX_POINTS = 60;

  let selectedCoin = initialCoin;
  let currentPrice: number | null = null;
  let previousPrice: number | null = null;
  let priceHistory: PricePoint[] = [];
  let lastUpdated: Date | null = null;
  let direction: "up" | "down" | "same" = "same";
  let loading = true;
  let errorMessage = "";
  let updateKey = 0;

  let pollingInterval: ReturnType<typeof setInterval> | undefined;
  let activeRequest: AbortController | null = null;
  let requestVersion = 0;

  const currencyFormatter = new Intl.NumberFormat("es-CL", {
    style: "currency",
    currency: "USD",
    minimumFractionDigits: 2,
    maximumFractionDigits: 8,
  });

  const coinLabel = (coin: string) =>
    coin
      .split("-")
      .map((part) => part.charAt(0).toUpperCase() + part.slice(1))
      .join(" ");

  const formatPrice = (price: number) => currencyFormatter.format(price);

  const buildChart = (history: PricePoint[]) => {
    if (history.length === 0) {
      return {
        points: "",
        min: 0,
        midpoint: 0,
        max: 0,
        startTime: null as Date | null,
        endTime: null as Date | null,
      };
    }

    const prices = history.map((point) => point.price);
    const min = Math.min(...prices);
    const max = Math.max(...prices);
    const range = max - min;

    const points = history
      .map((point, index) => {
        const x =
          history.length === 1 ? 50 : (index / (history.length - 1)) * 100;
        const y = range === 0 ? 20 : 35 - ((point.price - min) / range) * 30;
        return `${x},${y}`;
      })
      .join(" ");

    return {
      points,
      min,
      midpoint: min + range / 2,
      max,
      startTime: new Date(history[0].timestamp),
      endTime: new Date(history[history.length - 1].timestamp),
    };
  };

  const formatAxisPrice = (price: number) => {
    if (price > 0 && price < 1) return formatPrice(price);

    return new Intl.NumberFormat("es-CL", {
      style: "currency",
      currency: "USD",
      minimumFractionDigits: 0,
      maximumFractionDigits: 2,
    }).format(price);
  };

  const formatAxisTime = (date: Date | null) =>
    date?.toLocaleTimeString("es-CL", {
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
    }) ?? "—";

  $: chart = buildChart(priceHistory);

  const stopPolling = () => {
    if (pollingInterval) {
      clearInterval(pollingInterval);
      pollingInterval = undefined;
    }

    activeRequest?.abort();
    activeRequest = null;
    requestVersion += 1;
  };

  const fetchPrice = async () => {
    const coinAtRequest = selectedCoin;
    const versionAtRequest = requestVersion;

    activeRequest?.abort();
    const controller = new AbortController();
    activeRequest = controller;

    try {
      const params = new URLSearchParams({
        ids: coinAtRequest,
        vs_currencies: "usd",
        include_last_updated_at: "true",
      });
      const response = await fetch(
        `https://api.coingecko.com/api/v3/simple/price?${params}`,
        { signal: controller.signal },
      );

      if (!response.ok) {
        throw new Error(`La API respondió con estado ${response.status}`);
      }

      const data: PriceResponse = await response.json();
      const nextPrice = data[coinAtRequest]?.usd;

      if (typeof nextPrice !== "number") {
        throw new Error("La API no entregó un precio para esta moneda");
      }

      if (
        controller.signal.aborted ||
        versionAtRequest !== requestVersion ||
        coinAtRequest !== selectedCoin
      ) {
        return;
      }

      previousPrice = currentPrice;
      direction =
        previousPrice === null
          ? "same"
          : nextPrice > previousPrice
            ? "up"
            : nextPrice < previousPrice
              ? "down"
              : "same";
      currentPrice = nextPrice;

      const apiTimestamp = data[coinAtRequest]?.last_updated_at;
      const timestamp = apiTimestamp ? apiTimestamp * 1_000 : Date.now();
      lastUpdated = new Date(timestamp);
      priceHistory = [...priceHistory, { price: nextPrice, timestamp }].slice(
        -MAX_POINTS,
      );
      errorMessage = "";
      updateKey += 1;
    } catch (error) {
      if (error instanceof DOMException && error.name === "AbortError") return;

      errorMessage =
        error instanceof Error
          ? error.message
          : "No fue posible actualizar el precio";
    } finally {
      if (activeRequest === controller) activeRequest = null;
      if (versionAtRequest === requestVersion) loading = false;
    }
  };

  const startPolling = (coin: string) => {
    const normalizedCoin = coin.trim().toLowerCase();
    if (!normalizedCoin) return;

    stopPolling();
    selectedCoin = normalizedCoin;
    currentPrice = null;
    previousPrice = null;
    priceHistory = [];
    lastUpdated = null;
    direction = "same";
    errorMessage = "";
    loading = true;

    void fetchPrice();
    pollingInterval = setInterval(() => void fetchPrice(), POLLING_INTERVAL);
  };

  const handleCryptoChange = (event: Event) => {
    const coin = (event as CustomEvent<{ coin?: string }>).detail?.coin;
    if (typeof coin === "string" && coin !== selectedCoin) startPolling(coin);
  };

  onMount(() => {
    const latestSelection = (window as CryptoWindow).__selectedCrypto;
    startPolling(latestSelection ?? initialCoin);
  });
  onDestroy(stopPolling);
</script>

<!-- La isla Vue publica este evento global cuando cambia el selector. -->
<svelte:window on:crypto-changed={handleCryptoChange} />

<div class="space-y-5" aria-live="polite">
  <div class="flex flex-wrap items-center justify-between gap-3">
    <div>
      <p class="text-sm text-slate-500">Moneda monitoreada</p>
      <h4 class="text-xl font-bold text-slate-800">{coinLabel(selectedCoin)}</h4>
    </div>
    <span
      class="inline-flex items-center gap-2 rounded-full bg-emerald-50 px-3 py-1 text-xs font-semibold text-emerald-700"
    >
      <span class="live-dot h-2 w-2 rounded-full bg-emerald-500"></span>
      Polling cada 20 s
    </span>
  </div>

  {#if loading}
    <div class="flex min-h-56 items-center justify-center rounded-xl bg-slate-50">
      <p class="text-sm text-slate-500">Consultando el precio actual…</p>
    </div>
  {:else if currentPrice !== null}
    <div class="rounded-xl border border-slate-100 bg-slate-50 p-5">
      <p class="mb-1 text-sm font-medium text-slate-500">Precio actual (USD)</p>
      {#key updateKey}
        <p
          class:price-up={direction === "up"}
          class:price-down={direction === "down"}
          class="price-value text-3xl font-extrabold tracking-tight text-slate-900"
        >
          {formatPrice(currentPrice)}
          {#if direction === "up"}
            <span class="ml-1 text-base text-emerald-600" aria-label="Subió">▲</span>
          {:else if direction === "down"}
            <span class="ml-1 text-base text-rose-600" aria-label="Bajó">▼</span>
          {/if}
        </p>
      {/key}
      <p class="mt-2 text-xs text-slate-400">
        Última actualización:
        {lastUpdated?.toLocaleTimeString("es-CL") ?? "—"}
      </p>
    </div>

    <div>
      <div class="mb-2 flex items-center justify-between">
        <p class="text-sm font-semibold text-slate-700">Últimas mediciones</p>
        <p class="text-xs text-slate-400">{priceHistory.length}/{MAX_POINTS}</p>
      </div>
      <div class="min-h-60 overflow-hidden rounded-xl border border-slate-100 bg-white p-4">
        {#if priceHistory.length < 2}
          <div class="flex min-h-52 items-center justify-center text-xs text-slate-400">
            El gráfico aparecerá con la siguiente medición.
          </div>
        {:else}
          <div class="chart-layout">
            <p class="y-axis-title">Precio (USD)</p>
            <div class="y-axis-labels" aria-hidden="true">
              <span>{formatAxisPrice(chart.max)}</span>
              <span>{formatAxisPrice(chart.midpoint)}</span>
              <span>{formatAxisPrice(chart.min)}</span>
            </div>

            <div class="min-w-0">
              <svg
                class="chart-svg"
                viewBox="0 0 100 40"
                preserveAspectRatio="none"
                role="img"
                aria-label={`Precio desde ${formatAxisPrice(chart.min)} hasta ${formatAxisPrice(chart.max)}`}
              >
                <line x1="0" y1="5" x2="100" y2="5" class="chart-grid" />
                <line x1="0" y1="20" x2="100" y2="20" class="chart-grid" />
                <line x1="0" y1="35" x2="100" y2="35" class="chart-axis" />
                <line x1="0" y1="5" x2="0" y2="35" class="chart-axis" />
                <polyline points={chart.points} class="chart-line" />
              </svg>

              <div class="mt-1 flex justify-between text-xs text-slate-500">
                <time datetime={chart.startTime?.toISOString()}>
                  {formatAxisTime(chart.startTime)}
                </time>
                <time datetime={chart.endTime?.toISOString()}>
                  {formatAxisTime(chart.endTime)}
                </time>
              </div>
              <p class="mt-1 text-center text-xs font-medium text-slate-600">Tiempo</p>
            </div>
          </div>
        {/if}
      </div>
    </div>
  {/if}

  {#if errorMessage}
    <div class="rounded-lg border border-amber-200 bg-amber-50 px-4 py-3 text-sm text-amber-800">
      {errorMessage}. Se volverá a intentar automáticamente.
    </div>
  {/if}
</div>

<style>
  .live-dot {
    animation: pulse 1.6s ease-in-out infinite;
  }

  .price-up {
    animation: flash-up 0.7s ease-out;
  }

  .price-down {
    animation: flash-down 0.7s ease-out;
  }

  .chart-axis {
    stroke: #94a3b8;
    stroke-width: 0.7;
    vector-effect: non-scaling-stroke;
  }

  .chart-grid {
    stroke: #e2e8f0;
    stroke-dasharray: 2 2;
    stroke-width: 0.5;
    vector-effect: non-scaling-stroke;
  }

  .chart-line {
    fill: none;
    stroke: #4f46e5;
    stroke-linecap: round;
    stroke-linejoin: round;
    stroke-width: 1.6;
    vector-effect: non-scaling-stroke;
  }

  .chart-layout {
    display: grid;
    grid-template-columns: auto auto minmax(0, 1fr);
    gap: 0.5rem;
    align-items: stretch;
  }

  .chart-svg {
    display: block;
    width: 100%;
    height: 10rem;
  }

  .y-axis-title {
    align-self: start;
    display: flex;
    align-items: center;
    height: 10rem;
    color: #475569;
    font-size: 0.75rem;
    font-weight: 500;
    writing-mode: vertical-rl;
    transform: rotate(180deg);
  }

  .y-axis-labels {
    align-self: start;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    box-sizing: border-box;
    height: 10rem;
    padding-block: 1.25rem;
    color: #64748b;
    font-size: 0.7rem;
    text-align: right;
  }

  @keyframes pulse {
    0%,
    100% {
      opacity: 1;
      transform: scale(1);
    }
    50% {
      opacity: 0.45;
      transform: scale(0.75);
    }
  }

  @keyframes flash-up {
    0% {
      color: #059669;
      background-color: #d1fae5;
    }
  }

  @keyframes flash-down {
    0% {
      color: #e11d48;
      background-color: #ffe4e6;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .live-dot,
    .price-up,
    .price-down {
      animation: none;
    }
  }
</style>
