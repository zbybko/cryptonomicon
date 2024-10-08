<script>
export default {
  name: 'App',
  data() {
    return {
      searchQuery: '',
      tickers: [],
      sell: null,
      graph: [],
      tickerSymbols: [],
      filteredTickers: [],
      mounted: false,
      isAddButtonDis: false,
      tickerExist: false,
      filter: '',
      page: 1,
      maxItemsPerPage: 6,
      hasNextPage: true,
    };
  },
  mounted() {
    this.fetchTickers();
    const tickersStorage = localStorage.getItem('cryptonomicon-list');

    if (tickersStorage) {
      this.tickers = JSON.parse(tickersStorage);

      this.tickers.forEach(ticker => {
        this.getSubscribeTickers(ticker.symbol);
      });
    }
    const urlStorage = Object.fromEntries(new URL(window.location).searchParams.entries())

    if (urlStorage.filter) {
      this.filter = urlStorage.filter;
    }
    if (urlStorage.page) {
      this.page = parseInt(urlStorage.page);
    }
  },

  methods: {
    getSubscribeTickers(tickerSymbol) {
      setInterval(async () => {
        const f = await fetch(`https://min-api.cryptocompare.com/data/price?fsym=${tickerSymbol}&tsyms=EUR&api_key=51eb80d25b167611647f40bada38cf68e0d4868cd515da24c624adfa9cbdbd22`);
        const data = await f.json();
        this.tickers.find(t => t.symbol === tickerSymbol).price = data.EUR === undefined ? 0 : data.EUR > 1 ? data.EUR.toFixed(2) : data.EUR.toPrecision(2);
        if (this.sell?.symbol === tickerSymbol) {
          this.graph.push(data.EUR);
        }
      }, 2000);
    },
    add() {
      const currentTicker = {
        symbol: this.searchQuery,
        price: '0',
      };
      this.getSubscribeTickers(currentTicker.symbol);
      this.tickers.push(currentTicker);
      this.searchQuery = "";
      this.onInput();
      localStorage.setItem('cryptonomicon-list', JSON.stringify(this.tickers));
    },
    normilizeGraph() {
      const maxValue = Math.max(...this.graph);
      const minValue = Math.min(...this.graph);
      return this.graph.map(
          price => (price - minValue) * 100 / (maxValue - minValue)
      )
    },
    onInput() {
      if (this.searchQuery) {
        this.filteredTickers = this.tickerSymbols
            .filter(ticker => ticker.toLowerCase().includes(this.searchQuery.toLowerCase()))
            .slice(0, 4);
      } else {
        this.filteredTickers = this.tickerSymbols.slice(17, 21);
      }

      if (this.tickers.find(t => t.symbol.toLowerCase() === this.searchQuery.toLowerCase())) { // Проверка существует ли вводимый символ в массиве
        this.isAddButtonDis = true;
        this.tickerExist = true;
      } else {
        this.isAddButtonDis = false;
        this.tickerExist = false;
      }
    },
    async fetchTickers() {
      try {
        const response = await fetch('https://min-api.cryptocompare.com/data/all/coinlist?summary=true');
        const allDataTickers = await response.json();
        this.tickerSymbols = Object.keys(allDataTickers.Data);
        this.mounted = true;
        this.filteredTickers = this.tickerSymbols.slice(17, 21);
      } catch (error) {
        console.error('Ошибка при получении данных:', error);
      }
    },
    select(ticker) {
      this.sell = ticker;
      this.graph = [];
    },
    handleDelete(tickerToRemove) {
      this.tickers = this.tickers.filter(ticker => ticker !== tickerToRemove);
    },
    insert(symbol) {
      this.searchQuery = symbol;

      this.$nextTick(() => {
        this.$refs.searchInput.dispatchEvent(new Event('input'));
      })
    },
    filteredTickersCard() {
      const start = (this.page - 1) * this.maxItemsPerPage;
      const end = this.page * this.maxItemsPerPage;
      const filteredTickers = this.tickers.filter(ticker => ticker.symbol.toLowerCase().includes(this.filter.toLowerCase()));
      this.hasNextPage = filteredTickers.length > end;

      return filteredTickers.length > 0 ? filteredTickers.slice(start, end) : [];
    }
  },
  watch: {
    filter() {
      this.page = 1;

      history.pushState(null, document.title, `${window.location.origin}?filter=${this.filter}&page=${this.page}`);
    },
    page() {
      history.pushState(null, document.title, `${window.location.origin}?filter=${this.filter}&page=${this.page}`);
    }
  }
}
</script>

<template>
  <div class="container mx-auto flex flex-col items-center bg-gray-100 p-4">
    <div
        v-if="!mounted"
        class="fixed w-100 h-100 opacity-80 bg-purple-800 inset-0 z-50 flex items-center justify-center">
      <svg class="animate-spin -ml-1 mr-3 h-12 w-12 text-white" xmlns="http://www.w3.org/2000/svg" fill="none"
           viewBox="0 0 24 24">
        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
        <path class="opacity-75" fill="currentColor"
              d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
      </svg>
    </div>
    <div class="container">
      <section>
        <div class="flex">
          <div class="max-w-xs">
            <label for="wallet" class="block text-sm font-medium text-gray-700"
            >Тикер</label
            >
            <div class="mt-1 relative rounded-md shadow-md">
              <input
                  v-model="searchQuery"
                  @keyup.enter="add"
                  @input="onInput"
                  ref="searchInput"
                  type="text"
                  name="wallet"
                  id="wallet"
                  class="block w-full p-1 border-gray-300 text-gray-900 focus:outline-none focus:ring-gray-500 focus:border-gray-500 sm:text-sm rounded-md"
                  placeholder="Например DOGE"
              />
            </div>
            <div class="flex bg-white shadow-md p-1 rounded-mdshadow-md flex-wrap">
            <span
                v-for="symbol in filteredTickers"
                :key="symbol"
                @click="insert(symbol)"
                class="inline-flex items-center px-2 m-1 rounded-md text-xs font-medium bg-gray-300 text-gray-800 cursor-pointer"
            >
                {{ symbol }}
            </span>
            </div>
            <div
                v-if="tickerExist"
                class="text-sm text-red-600">Такой тикер уже добавлен
            </div>
          </div>
        </div>
        <button
            :disabled="isAddButtonDis"
            @click="add"
            type="button"
            class="my-4 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
        >
          <!-- Heroicon name: solid/mail -->
          <svg
              class="-ml-0.5 mr-2 h-6 w-6"
              xmlns="http://www.w3.org/2000/svg"
              width="30"
              height="30"
              viewBox="0 0 24 24"
              fill="#ffffff"
          >
            <path
                d="M13 7h-2v4H7v2h4v4h2v-4h4v-2h-4V7zm-1-5C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8z"
            ></path>
          </svg>
          Добавить
        </button>
      </section>

      <template v-if="tickers.length > 0">
        <hr class="w-full border-t border-gray-600 my-4"/>
        <div>
          <div
              class="mb-2"
          >
            Фильтр:
          </div>
          <input
              v-model="filter"
              class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
          />
          <button
              v-if="page > 1"
              @click="page = page - 1"
              class="my-4 mx-2 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500">
            Назад
          </button>
          <button
              v-if="hasNextPage"
              @click="page = page + 1"
              class="my-4 mx-2 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500">
            Вперед
          </button>
        </div>
        <hr class="w-full border-t border-gray-600 my-4"/>
        <dl class="mt-5 grid grid-cols-1 gap-5 sm:grid-cols-3">
          <div
              v-for="t in filteredTickersCard()"
              :key="t.symbol"
              @click="select(t)"
              :class="{
                'border-4': sell === t
              }"
              class="bg-white overflow-hidden shadow rounded-lg border-purple-800 border-solid cursor-pointer"
          >
            <div class="px-4 py-5 sm:p-6 text-center">
              <dt class="text-sm font-medium text-gray-500 truncate">
                {{ t.symbol }} - USD
              </dt>
              <dd class="mt-1 text-3xl font-semibold text-gray-900">
                {{ t.price }}
              </dd>
            </div>
            <div class="w-full border-t border-gray-200"></div>
            <button
                @click.stop="handleDelete(t)"
                class="flex items-center justify-center font-medium w-full bg-gray-100 px-4 py-4 sm:px-6 text-md text-gray-500 hover:text-gray-600 hover:bg-gray-200 hover:opacity-20 transition-all focus:outline-none"
            >
              <svg
                  class="h-5 w-5"
                  xmlns="http://www.w3.org/2000/svg"
                  viewBox="0 0 20 20"
                  fill="#718096"
                  aria-hidden="true"
              >
                <path
                    fill-rule="evenodd"
                    d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z"
                    clip-rule="evenodd"
                ></path>
              </svg>
              Удалить
            </button>
          </div>
        </dl>
        <hr class="w-full border-t border-gray-600 my-4"/>
      </template>
      <section class="relative" v-if="sell">
        <h3 class="text-lg leading-6 font-medium text-gray-900 my-8">
          {{ sell.symbol }} - USD
        </h3>
        <div class="flex items-end border-gray-600 border-b border-l h-64">
          <div
              v-for="(bar, idx) in normilizeGraph()"
              :key="idx"
              :style="{ height: `${bar}%` }"
              class="bg-purple-800 border w-1"
          ></div>
        </div>
        <button
            type="button"
            class="absolute top-0 right-0"
            @click="sell = null"
        >
          <svg
              xmlns="http://www.w3.org/2000/svg"
              xmlns:xlink="http://www.w3.org/1999/xlink"
              version="1.1"
              width="30"
              height="30"
              x="0"
              y="0"
              viewBox="0 0 511.76 511.76"
              style="enable-background:new 0 0 512 512"
              xml:space="preserve"
          >
          <g>
            <path
                d="M436.896,74.869c-99.84-99.819-262.208-99.819-362.048,0c-99.797,99.819-99.797,262.229,0,362.048    c49.92,49.899,115.477,74.837,181.035,74.837s131.093-24.939,181.013-74.837C536.715,337.099,536.715,174.688,436.896,74.869z     M361.461,331.317c8.341,8.341,8.341,21.824,0,30.165c-4.16,4.16-9.621,6.251-15.083,6.251c-5.461,0-10.923-2.091-15.083-6.251    l-75.413-75.435l-75.392,75.413c-4.181,4.16-9.643,6.251-15.083,6.251c-5.461,0-10.923-2.091-15.083-6.251    c-8.341-8.341-8.341-21.845,0-30.165l75.392-75.413l-75.413-75.413c-8.341-8.341-8.341-21.845,0-30.165    c8.32-8.341,21.824-8.341,30.165,0l75.413,75.413l75.413-75.413c8.341-8.341,21.824-8.341,30.165,0    c8.341,8.32,8.341,21.824,0,30.165l-75.413,75.413L361.461,331.317z"
                fill="#718096"
                data-original="#000000"
            ></path>
          </g>
        </svg>
        </button>
      </section>
    </div>
  </div>
</template>

<style>

</style>
