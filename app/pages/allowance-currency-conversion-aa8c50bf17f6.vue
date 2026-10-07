<template>
  <div class="min-h-screen bg-zinc-950 text-slate-100">
    <div class="mx-auto w-full max-w-3xl px-6 py-16 sm:px-12 lg:py-24">
      <header>
        <h1 class="text-3xl font-bold tracking-tight text-white sm:text-4xl">
          Allowance Currency Conversion
        </h1>
        <p class="mt-3 text-sm text-slate-400">
          €{{ formatNumber(EUR_AMOUNT, 0) }} EUR → USD at the average of two mid-market rates, plus a
          ${{ formatNumber(TRANSFER_FEE_USD, 0) }} transfer fee.
        </p>
      </header>

      <section class="mt-12">
        <div v-if="status === 'loading'" class="text-slate-400">Fetching mid-market rates…</div>

        <div v-else-if="status === 'error'" class="space-y-3">
          <p class="text-red-400">{{ errorMessage }}</p>
          <button
            type="button"
            class="cursor-pointer text-sm text-slate-300 underline underline-offset-4 hover:text-white"
            @click="load"
          >
            Try again
          </button>
        </div>

        <div v-else-if="result">
          <p class="text-xl tabular-nums text-slate-300 sm:text-2xl">
            {{ formatNumber(EUR_AMOUNT, 0) }} EUR = ${{ formatNumber(result.converted, 0) }} USD
          </p>
          <p class="mt-6 text-sm font-medium uppercase tracking-widest text-slate-400">Amount to send</p>
          <p class="mt-2 text-5xl font-bold tabular-nums tracking-tight text-white sm:text-6xl">
            AMOUNT TO SEND: ${{ formatNumber(result.final, 0) }}
          </p>
          <p class="mt-4 text-sm text-slate-400">
            Calculated {{ formatDateTime(result.fetchedAt) }}
          </p>
          <button
            type="button"
            class="mt-2 cursor-pointer text-sm text-slate-300 underline underline-offset-4 hover:text-white"
            @click="load"
          >
            Refresh rates
          </button>
        </div>
      </section>

      <section v-if="status === 'ready' && result" class="mt-16 border-t border-white/10 pt-8">
        <h2 class="text-base font-semibold text-white">Order of operations</h2>
        <dl class="mt-5 space-y-3 font-mono text-sm tabular-nums">
          <div
            v-for="(rate, i) in result.rates"
            :key="rate.source"
            class="flex flex-wrap justify-between gap-x-4 gap-y-1"
          >
            <dt class="text-slate-400">
              Mid market {{ i + 1 }}
              <span class="text-slate-500">({{ rate.source }}, rate date {{ rate.date }})</span>
            </dt>
            <dd class="text-white">1 EUR = {{ formatNumber(rate.rate, 6) }} USD</dd>
          </div>
          <div class="flex flex-wrap justify-between gap-x-4 gap-y-1">
            <dt class="text-slate-400">Average</dt>
            <dd class="text-white">
              ({{ formatNumber(result.rates[0].rate, 6) }} + {{ formatNumber(result.rates[1].rate, 6) }}) ÷ 2
              = {{ formatNumber(result.average, 6) }}
            </dd>
          </div>
          <div class="flex flex-wrap justify-between gap-x-4 gap-y-1">
            <dt class="text-slate-400">USD conversion</dt>
            <dd class="text-white">
              {{ formatNumber(EUR_AMOUNT, 0) }} EUR × {{ formatNumber(result.average, 6) }}
              = ${{ formatNumber(result.exact, 2) }} USD
            </dd>
          </div>
          <div class="flex flex-wrap justify-between gap-x-4 gap-y-1">
            <dt class="text-slate-400">Rounded down</dt>
            <dd class="text-white">
              {{ formatNumber(EUR_AMOUNT, 0) }} EUR = ${{ formatNumber(result.converted, 0) }} USD
            </dd>
          </div>
          <div class="flex flex-wrap justify-between gap-x-4 gap-y-1">
            <dt class="text-slate-400">Transfer fee</dt>
            <dd class="text-white">+ ${{ formatNumber(TRANSFER_FEE_USD, 0) }}</dd>
          </div>
          <div class="flex flex-wrap justify-between gap-x-4 gap-y-1 border-t border-white/10 pt-3">
            <dt class="font-semibold text-white">Final amount</dt>
            <dd class="font-semibold text-white">${{ formatNumber(result.final, 0) }}</dd>
          </div>
        </dl>
      </section>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from "vue";

const EUR_AMOUNT = 2450;
const TRANSFER_FEE_USD = 5;
const TIMEOUT_MS = 8000;

useHead({
  title: "Allowance Conversion",
  meta: [
    { name: "robots", content: "noindex, nofollow, noarchive, nosnippet, noimageindex, noai, noimageai" },
    { name: "googlebot", content: "noindex, nofollow, noarchive, nosnippet" },
    { name: "bingbot", content: "noindex, nofollow, noarchive, nosnippet" },
  ],
});

type Rate = { source: string; rate: number; date: string };
type Result = {
  rates: [Rate, Rate];
  average: number;
  exact: number;
  converted: number;
  final: number;
  fetchedAt: Date;
};

// Free, keyless, CORS-enabled sources. The first two are primary; the rest are
// fallbacks used only if a primary fails, so we always end up with two rates.
const sources: { name: string; fetch: () => Promise<Rate> }[] = [
  {
    name: "Frankfurter (ECB)",
    fetch: async () => {
      const data = await getJson("https://api.frankfurter.dev/v1/latest?base=EUR&symbols=USD");
      return { source: "Frankfurter (ECB)", rate: data.rates.USD, date: data.date };
    },
  },
  {
    name: "ExchangeRate-API",
    fetch: async () => {
      const data = await getJson("https://open.er-api.com/v6/latest/EUR");
      if (data.result !== "success") throw new Error("ExchangeRate-API returned an error");
      return {
        source: "ExchangeRate-API",
        rate: data.rates.USD,
        date: new Date(data.time_last_update_unix * 1000).toISOString().slice(0, 10),
      };
    },
  },
  {
    name: "fawazahmed0 currency-api",
    fetch: async () => {
      const urls = [
        "https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@latest/v1/currencies/eur.min.json",
        "https://latest.currency-api.pages.dev/v1/currencies/eur.min.json",
      ];
      let lastError: unknown;
      for (const url of urls) {
        try {
          const data = await getJson(url);
          return { source: "fawazahmed0 currency-api", rate: data.eur.usd, date: data.date };
        } catch (e) {
          lastError = e;
        }
      }
      throw lastError;
    },
  },
];

const status = ref<"loading" | "ready" | "error">("loading");
const errorMessage = ref("");
const result = ref<Result | null>(null);

async function getJson(url: string) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), TIMEOUT_MS);
  try {
    const res = await fetch(url, { signal: controller.signal, cache: "no-store" });
    if (!res.ok) throw new Error(`${url} responded ${res.status}`);
    return await res.json();
  } finally {
    clearTimeout(timer);
  }
}

function isValidRate(r: Rate) {
  return typeof r.rate === "number" && Number.isFinite(r.rate) && r.rate > 0.5 && r.rate < 2;
}

async function load() {
  status.value = "loading";
  errorMessage.value = "";

  // Fire everything in parallel, then take the first two valid rates in priority order.
  const settled = await Promise.allSettled(sources.map((s) => s.fetch()));
  const rates = settled
    .filter((s): s is PromiseFulfilledResult<Rate> => s.status === "fulfilled")
    .map((s) => s.value)
    .filter(isValidRate)
    .slice(0, 2);

  if (rates.length < 2) {
    status.value = "error";
    errorMessage.value = `Only got ${rates.length} of 2 required rates. Check your connection and try again.`;
    return;
  }

  const [a, b] = rates as [Rate, Rate];
  const average = (a.rate + b.rate) / 2;
  const exact = EUR_AMOUNT * average;
  // Round down to the nearest whole dollar before adding the fee.
  const converted = Math.floor(exact);
  const final = converted + TRANSFER_FEE_USD;

  result.value = { rates: [a, b], average, exact, converted, final, fetchedAt: new Date() };
  status.value = "ready";
}

function formatNumber(n: number, digits: number) {
  return n.toLocaleString("en-US", { minimumFractionDigits: digits, maximumFractionDigits: digits });
}

function formatDateTime(d: Date) {
  return d.toLocaleString("en-US", {
    dateStyle: "full",
    timeStyle: "long",
  });
}

onMounted(load);
</script>
