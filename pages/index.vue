<template>
  <div>
    <div class="container-fluid">
      <div class="row accordion" role="tablist">
        <b-card v-for="category in goodsByCategory" :key="category.id" no-body class="col-md-6 mb-1">
          <b-card-header header-tag="header" class="p-1" role="tab">
            <b-button v-b-toggle="`category-${category.id}`" block variant="info">{{ category.name }}</b-button>
          </b-card-header>
          <b-collapse :id="`category-${category.id}`" role="tabpanel">
            <b-card-body>
              <div class="row">
                <div v-for="good in category.goods" :key="good.id" class="col-md-6 mb-1 row">
                  <div class="col-md-9 good-name">
                    {{ good.name }} ({{ good.stock }})
                    <button class="btn btn-link" @click="addToCart(good.id)">
                      <b-icon-cart/>
                    </button>
                  </div>
                  <div class="col-md-3 border-left good-price"
                       :class="{'bg-danger': good.priceChange === 0, 'bg-success': good.priceChange === 1}">
                    {{ priceByCurrency(good.price) }}
                  </div>
                </div>
              </div>
            </b-card-body>
          </b-collapse>
        </b-card>
      </div>
      <div v-show="showCart" class="row mt-3">
        <div class="col-12">
          <div class="card m-b-30">
            <div class="card-header">
              <h5 class="card-title">Корзина</h5>
            </div>
            <b-alert
              v-model="showAlert"
              dismissible
              fade
              variant="warning"
            >{{ alertText }}
            </b-alert>
            <div class="card-body">
              <div class="row justify-content-center">
                <div class="col-md-12">
                  <div class="cart-container">
                    <div class="cart-head">
                      <div class="table-responsive">
                        <table class="table table-borderless">
                          <thead>
                          <tr>
                            <th scope="col">Товар</th>
                            <th scope="col">Количество</th>
                            <th scope="col">Стоимость</th>
                            <th scope="col"></th>
                          </tr>
                          </thead>
                          <tbody>
                          <tr v-for="item in cartItems.items" :key="item.id">
                            <td>{{ item.name }}</td>
                            <td>
                              <input v-model.number="cart[item.id].qty" type="number" class="form-control"
                                     :max="item.stock" :min="0"/>
                            </td>
                            <td class="text-right">{{ item.price }}</td>
                            <td class="text-right">
                              <button class="btn btn-link" @click="updateQty(item.id, 0)">Удалить</button>
                            </td>
                          </tr>
                          </tbody>
                        </table>
                      </div>
                    </div>
                    <div class="cart-body">
                      <div class="row">
                        <div class="col-md-12 text-right">
                          <div class="order-total table-responsive ">
                            Итого: {{ cartItems.totalPrice }}
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="col-md-6">
        Курс:
        <b-input v-model="course" type="number" :min="courseMin" :max="courseMax"/>
      </div>
    </div>
  </div>
</template>

<script>
import _ from 'lodash';
import {BIconCart} from 'bootstrap-vue'

export default {
  components: {
    BIconCart
  },
  async asyncData({$content, params}) {
    // получаем данные из nuxtjs content
    const data = await $content('data').fetch()
    const names = await $content('names').fetch()
    return {data, names}
  },
  data() {
    return {
      showAlert: false,
      alertText: '',
      dataTimer: '',
      courseMin: 20,
      courseMax: 20,
      course: 20,
      cart: {},
    }
  },
  computed: {
    // Подготавливаем товары для вывода в каталог
    goods() {
      const goods = _.filter(this.data.Value.Goods, (good) => {
        return this.formattedGood(good) !== null
      })
      return _.map(goods, (good) => {
        return this.formattedGood(good)
      });
    },
    // Разбиваем товары по группам
    goodsByCategory() {
      const goods = _.groupBy(this.goods, 'category');
      return _.map(goods, (goods, categoryId) => {
        return {
          id: categoryId,
          name: this.names[categoryId].G,
          goods
        }
      })
    },
    // Проверяем наличие товаров в корзине
    showCart() {
      return !_.isEmpty(this.cart)
    },
    // Подготавливаем товары, добавленные в корзину для вывода в корзине
    cartItems() {
      const cartItems = {
        items: {},
        totalPrice: 0
      }
      cartItems.items = _.map(this.cart, (item) => {
        const good = _.find(this.goods, (good) => {
          return good.id === item.id
        })

        const price = this.priceByCurrency(good.price);
        cartItems.totalPrice += price * item.qty
        return {
          id: good.id,
          qty: item.qty > good.stock ? good.stock : item.qty,
          name: good.name,
          price,
          stock: good.stock,
        }
      })

      return cartItems;
    },
  },
  watch: {
    // отслеживаем изменение цены товара
    goods: {
      deep: true,
      handler(newVal, oldVal) {
        _.forOwn(newVal, (good) => {
            const oldGood = _.find(oldVal, (oldGood) => {
              if (typeof (good) !== 'undefined' && typeof (oldGood) !== 'undefined') {
                return oldGood.id === good.id
              }
            })
            if (typeof (oldGood) !== 'undefined') {
              const newPrice = this.priceByCurrency(good.price)
              const oldPrice = this.priceByCurrency(oldGood.price)
              if (newPrice > oldPrice) {
                good.priceChange = 1
              } else if (newPrice < oldPrice) {
                good.priceChange = 0
              } else {
                good.priceChange = null
              }
            }
          }
        )
      }
    },
    // отслеживаем кол-во добавленного товара в корзину и его наличие
    cart: {
      deep: true,
      handler(cart) {
        _.forOwn(cart, (item) => {
          if (item.qty <= 0) {
            this.removeFromCart(item.id)
          }
          const good = _.find(this.goods, (good) => {
            return good.id === item.id
          })
          if (good.stock < item.qty) {
            item.qty = good.stock
            this.sendAlert('Товара больше нет на складе');
          }
        })
      }
    }
    ,
// проверяем минимальное значение для курса валюты
    course: {
      handler(course) {
        if (parseInt(course) < this.courseMin) {
          this.course = this.courseMin
        } else if (parseInt(course) > this.courseMax) {
          this.course = this.courseMax
        }
      }
    }
  },
  methods: {
    formattedGood(good) {
      if (typeof (this.names[good.G]) === 'undefined' || typeof (this.names[good.G].B[good.T]) === 'undefined') {
        return null
      }
      const goodName = this.names[good.G].B[good.T]
      return {
        price: good.C,
        priceChange: null,
        id: good.T,
        category: good.G,
        name: goodName.N,
        stock: good.P,
      }

    }
    ,
    priceByCurrency(price) {
      return Math.round(parseFloat(price) * parseFloat(this.course));
    }
    ,
    checkGoodQty(good) {
      const qty = this.cart[good.id] ? this.cart[good.id].qty : 0
      if (qty === good.stock) {
        this.sendAlert('Товара больше нет на складе');
        return false;
      }

      return true;
    }
    ,
    addToCart(id) {
      if (this.checkGoodQty(id)) {
        if (this.cart[id]) {
          const newQty = this.cart[id].qty + 1;
          this.updateQty(id, newQty)
        } else {
          this.$set(this.cart, id, {id, qty: 1})
        }
      }
    }
    ,
    updateQty(id, qty) {
      this.cart[id].qty = parseInt(qty)
    }
    ,
    removeFromCart(id) {
      this.$delete(this.cart, id)
    }
    ,
    sendAlert(text) {
      this.alertText = text;
      this.showAlert = true;
    }
    ,
  }
}
</script>
<style scoped>
.good-name {
  font-size: .6em;
}

.good-price {
  font-size: .6em;
}

.cart-container {
  border: 1px solid rgba(0, 0, 0, 0.05);
  padding: 30px;
}

.cart-container .cart-body {
  border-top: 1px solid rgba(0, 0, 0, 0.05);
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
  padding: 30px 0 20px;
  margin: 20px 0 30px;
}
</style>
