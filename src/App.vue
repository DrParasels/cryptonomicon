<template>
  <div class="container mx-auto flex flex-col items-center bg-gray-100 p-4">
    <div class="container">
      <add-ticker
        @add-ticker="add"
        @filter-ticker="test"
        :disabled="tooManyTickers"
        :coinList="coinList"
        :flag="flag"
      />
      <template v-if="tickers.length">
        <hr class="w-full border-t border-gray-600 my-4" />
        <div>
          <div>Фильтр: <input v-model="filter" type="text" /></div>
          <button
            @click="page = page - 1"
            v-if="this.page > 1"
            class="my-4 mx-2 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
          >
            Назад
          </button>
          <button
            @click="page = page + 1"
            v-if="hasNextPage"
            class="my-4 mx-2 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
          >
            Вперед
          </button>
        </div>
        <hr class="w-full border-t border-gray-600 my-4" />
        <dl class="mt-5 grid grid-cols-1 gap-5 sm:grid-cols-3">
          <div
            v-for="t of paginatedTickers"
            :key="t.name"
            @click="select(t)"
            :class="{
              'border-4': selectedTicker === t,
              'bg-red-100': noCurrencyTickers.includes(t.name)
            }"
            class="bg-white overflow-hidden shadow rounded-lg border-purple-800 border-solid cursor-pointer"
          >
            <div class="px-4 py-5 sm:p-6 text-center">
              <dt class="text-sm font-medium text-gray-500 truncate">
                {{ t.name }} - USD
              </dt>
              <dd class="mt-1 text-3xl font-semibold text-gray-900">
                {{ formatPrice(t.price) }}
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
        <hr class="w-full border-t border-gray-600 my-4" />
      </template>
      <graph-section
        v-if="selectedTicker"
        :ticker-price="selectedTicker.price"
        :ticker-name="selectedTicker.name"
        @close-graph="closeGraph"
      />
    </div>
  </div>
</template>

<script>
// [x] 5. Наличие в состоянии ЗАВИСИМЫХ ДАННЫХ | Критичность: 5+
// [] 6. Запросы напрямую внутри компонента | Критичность: 5
// [x] 2. При удалении остаётся подписка на загрузку тикера | Критичность: 5 (вроде бы пофиксил)
// [] 4. Обработка ошибок API | Критичность: 5
// [x] 3. Количество запросов | Критичность: 4
// [x] 8. При удалении тикера не изменяется localStorage | Критичность: 4 (вроде бы пофиксил)
// [x] 1. Одинаковый код в watch | Критичность: 3
// [x] 9. localStorage и анонимные вкладки | Критичность: 3
// [x] 7. График ужасно выглядит, если будет много цен | Критичность: 2
// [] 10. Магические строки и числа (URL, 5000милисекунд задежки, ключ локал сторейдж) | Критичность: 1

import { subscribeToTicker, unsubscribeFromTicker, noDataTickers } from "./api";
import AddTicker from "@/components/AddTicker.vue";
import GraphSection from "@/components/GraphSection.vue";

export default {
  name: "App",

  components: {
    AddTicker,
    GraphSection
  },

  data() {
    return {
      ticker: "",
      filter: "",
      maxGraphElements: 1,
      tickers: [],
      selectedTicker: null,
      noCurrencyTickers: [],
      graph: [],
      flag: "",
      coinList: [],
      arr: [],
      page: 1
    };
  },

  created() {
    setTimeout(async () => {
      const f = await fetch(
        `https://min-api.cryptocompare.com/data/all/coinlist?summary=true`
      );
      let result = await f.json();
      for (let item in result.Data) {
        this.arr.push(item);
      }
    }, 0);

    const windowData = Object.fromEntries(
      new URL(window.location).searchParams.entries()
    );
    if (windowData.filter) {
      this.filter = windowData.filter;
    }

    if (windowData.page) {
      this.page = windowData.page;
    }
    const tickersData = localStorage.getItem("cryptonomicon-list");

    if (tickersData) {
      this.tickers = JSON.parse(tickersData);
      this.tickers.forEach((ticker) => {
        subscribeToTicker(ticker.name, (newPrice) => {
          this.updateTicker(ticker.name, newPrice);
          noDataTickers.forEach((x) => this.noCurrencyTickers.push(x));
        });
      });
      this.noCurrencyTickers.forEach((item) => {
        this.notHaveDataTicker = item;
      });
    }

    setInterval(this.updateTickers, 5000);
  },

  mounted() {
    window.addEventListener("resize", this.calculateMaxGraphElements);
  },

  beforeUnmount() {
    window.removeEventListener("resize", this.calculateMaxGraphElements);
  },

  computed: {
    tooManyTickers() {
      return this.tickers.length > 4;
    },
    startIndex() {
      return (this.page - 1) * 6;
    },

    endIndex() {
      return this.page * 6;
    },
    filteredTickers() {
      return this.tickers.filter((ticker) =>
        ticker.name.includes(this.filter.toUpperCase())
      );
    },

    paginatedTickers() {
      return this.filteredTickers.slice(this.startIndex, this.endIndex);
    },

    hasNextPage() {
      return this.filteredTickers.length > this.endIndex;
    },

    pageStateOptions() {
      return {
        filter: this.filter,
        page: this.page
      };
    }
  },

  methods: {
    closeGraph() {
      this.selectedTicker = null;
    },
    formatPrice(price) {
      if (price === "-") {
        return price;
      }
      return price > 1 ? price.toFixed(2) : price.toPrecision(2);
    },
    test(ticker) {
      this.flag = "";
      const result = this.arr.filter((word) =>
        word.match(ticker.toUpperCase())
      );

      if (result.length > 4) {
        let a = result.length - 4;
        result.splice(4, a);
      }
      this.coinList = result;
    },
    add(ticker) {
      if (this.coinList.length === 0) {
        this.flag = "Такого тикера не существует";
      } else {
        const currentTicker = {
          name: ticker,
          price: "-"
        };
        if (this.tickers.find((i) => i.name === ticker) !== undefined) {
          this.flag = "Такой тикер уже добавлен";
        } else {
          this.tickers = [...this.tickers, currentTicker];
          this.filter = "";
          subscribeToTicker(currentTicker.name, (newPrice) => {
            this.updateTicker(currentTicker.name, newPrice);
          });
        }
      }
    },
    updateTicker(tickerName, price) {
      this.tickers
        .filter((t) => t.name === tickerName)
        .forEach((t) => {
          t.price = price;
        });
    },
    select(ticker) {
      this.selectedTicker = ticker;
      console.log(ticker);
    },
    handleDelete(tickerToRemove) {
      this.tickers = this.tickers.filter((t) => t != tickerToRemove);
      this.selectedTicker =
        tickerToRemove === this.selectedTicker ? null : this.selectedTicker;
      unsubscribeFromTicker(tickerToRemove.name);
    }
  },

  watch: {
    tickers() {
      localStorage.setItem("cryptonomicon-list", JSON.stringify(this.tickers));
    },
    paginatedTickers() {
      if (this.paginatedTickers.length === 0 && this.page > 1) {
        this.page -= 1;
      }
    },
    filter() {
      this.page = 1;
    },
    pageStateOptions(vlaue) {
      window.history.pushState(
        null,
        document.title,
        `${window.location.pathname}?filter=${vlaue.filter}&page=${vlaue.page}`
      );
    }
  }
};
</script>
