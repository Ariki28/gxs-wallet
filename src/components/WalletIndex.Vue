<template>
  <div class="page-group">
    <div class="page" id="page-wallet-index">
      <div ref="bar" class="bar bar-nav buttons-fixed" style="background: rgba(37, 40, 113, 0)">
        <a href="javascript:;" class="icon gxicon gxicon-user pull-left" @click="openPanel"></a>
        <a v-if="isNative" href="javascript:;" class="icon gxicon gxicon-scan pull-right" @click="openQRScaner">
          <input ref="qrfile" @change="onFileUpload" v-if="!isNative" type="file" class="file-selector"/>
        </a>
      </div>
      <div ref="bg" id="bg"></div>
      <div class="content pull-to-refresh-content" ref="content">
        <div class="row-top" ref="top" :class="{ios:$route.query.platform=='ios'}">
          <div class="pull-to-refresh-layer">
            <div class="preloader">
              <div class="line-scale">
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
              </div>
            </div>
          </div>
          <div class="loading" v-if="wallets.length==0">
            <div class="line-scale">
              <div></div>
              <div></div>
              <div></div>
              <div></div>
              <div></div>
            </div>
          </div>
          <!--wallets-->
          <swiper :options="swiperOption" :not-next-tick="true" ref="mySwiper" v-show="wallets.length>0">
            <swiper-slide v-for="wallet in wallets" :key="wallet.name">
              <div class="wallet-info">
                <div class="text-center">
                  <account-image :account="wallet.account" :size="30"></account-image>
                  <div>
                    <a @click="openQRCodeModal" href="javascript:;" class="link-qrcode">{{wallet.account}}&nbsp;
                      <small class="icon icon-code"></small>
                    </a>
                  </div>
                  <p class="bak-tip">
                    <router-link :to="linkBackup(wallet.account)" class="btn-backup"
                                 :class="{visible:!wallet.backup_date}" href="javascript:;">
                      <small>{{$t('index.backup_wallet')}}&nbsp;<span class="red-dot"></span><span
                        class="icon icon-right"></span></small>
                    </router-link>
                  </p>
                </div>
                <div class="row-balance">
                  <div class="item">
                    <small>{{$t('index.asset_name')}}</small>
                    <div class="digit balance">{{wallet.balance | asset(2)}}</div>
                  </div>
                  <div class="item item-middle">≈</div>
                  <div class="item value">
                    <small>{{$t('index.value')}}</small>
                    <div class="digit balance">{{getMarketCap(wallet) | number(2)}}</div>
                  </div>
                </div>
              </div>
            </swiper-slide>
            <div class="swiper-pagination" slot="pagination" v-show="wallets&&wallets.length>1"></div>
          </swiper>
          <router-link :to="link('/loyalty-program')" class="link-loyalty-program">
            <img :src="`static/${$t('index.loyalty_program_icon')}`">
          </router-link>
        </div>
        <!--function entries-->
        <div class="row-operation">
          <router-link :to="link('/transfer')" class="item"
                       href="javascript:;">
            <span class="gxicon x2 gxicon-transfer"></span><span class="item-text">{{$t('index.transfer')}}</span>
          </router-link>
          <div class="sep"></div>
          <a @click="openQRCodeModal" class="item" href="javascript:;">
            <span class="gxicon x2 gxicon-collect"></span><span class="item-text">{{$t('index.receive')}}</span>
          </a>
        </div>
        <!--exchange list-->
         <div class="table-exchanges">
          <div class="list-block media-list">
            <ul>
              <li v-for="exchange in exchanges">
                <!-- <router-link :to="`/realtime-quotations/${exchange.name}/${exchange.symbol}`"> -->
                  <div class="item-content item-exchange">
                    <div class="item-inner">
                      <div class="title-ex-name">
                        <div class="asset-name">{{exchange.symbol}}</div>
                        <div class="ex-name">{{exchange.name}}</div>
                      </div>
                      <div class="price">
                        <div class="price_primary">
                          {{exchange.price}}
                        </div>
                        <div class="price_secondary">
                          {{_i18n.locale == 'zh-CN' ? `￥${exchange.price_rmb}` : `$${exchange.price_dollar}`}}
                        </div>
                      </div>
                      <div class="quote" :class="{green:exchange.quote>0}">
                        {{(exchange.quote > 0 ? '+' : '') + (exchange.quote).toFixed(2) + '%'}}
                      </div>
                    </div>
                  </div>
                <!-- </router-link> -->
              </li>
            </ul>
          </div>
        </div>
      </div>
    </div>
    <left-panel ref="panel"></left-panel>
    <account-q-r-code ref="qrcode" :account="this.currentWallet.account"></account-q-r-code>
  </div>
</template>
<script>
  import {swiper, swiperSlide} from 'vue-awesome-swiper'
  import AccountImage from './sub/AccountImage.vue'
  import LeftPanel from './sub/LeftPanel.vue'
  import {ChainStore} from 'gxbjs'
  import AccountQRCode from './sub/AccountQRCode.vue'
  import {get_market_info} from '@/services/MarketService'
  import util from '@/common/util'

  import {
    get_wallets,
    fetch_account_balance,
    get_wallet_index,
    set_wallet_index
  } from '@/services/WalletService'
  import filters from '@/filters'

  export default {
    filters,
    data() {
      let self = this;
      let wallets = get_wallets();
      wallets = wallets.map((w) => {
        w.balance = this.$t('index.balance_loading');
        return w;
      });
      let wallet_index = get_wallet_index();
      return {
        accountCopied: false,
        stopFetchingMarket: false,
        wallets: wallets,
        currentWalletIndex: wallet_index,
        exchanges: [],
        swiperOption: {
          pagination: '.swiper-pagination',
          autoplay:false,
          keyboardControl: true,
          paginationClickable: true,
          initialSlide: wallet_index,
          onSlideChangeEnd: function (s) {
            set_wallet_index(s.activeIndex);
            self.currentWalletIndex = s.activeIndex;
          }
        }
      }
    },
    watch: {
      currentWalletIndex() {
        this.loadBalance(this.currentWallet);
      }
    },

    methods: {
//      goLoyaltyProgram() {
//        this.$router.push({path: this.link('/loyalty-program')})
//      },
      loadMarketInfo() {
        get_market_info().then((resp) => {
          this.exchanges = resp;
          setTimeout(() => {
            if (!this.stopFetchingMarket) {
              this.loadMarketInfo();
            }
          }, 3000);
        }).catch(ex => {
          console.error(ex);
          setTimeout(() => {
            if (!this.stopFetchingMarket) {
              this.loadMarketInfo();
            }
          }, 3000);
        })
      },
      loadWallets() {
        if (this.wallets.length == 0) {
          this.$router.replace({
            path: this.link('/wallet-create')
          });
        }
        else {
          this.wallets.forEach((wallet) => {
            this.loadBalance(wallet);
          })
        }
        setTimeout(() => {
          $.pullToRefreshDone($(this.$el).find('.pull-to-refresh-content'));
        }, 500)
      },
      loadBalance(wallet) {
        if (wallet.account) {
          fetch_account_balance(wallet.account).then(function (balance) {
            wallet.balance = balance.amount / 100000;
          }).catch(ex => {
            console.error(ex);
          })
        }
      },
      getMarketCap(wallet) {
        let balance = Number(wallet.balance);
        let price = this.price;
        if (!isNaN(balance) && price != '-') {
          return wallet.balance * this.price;
        }
        return '-';
      },
      openPanel() {
        this.$refs.panel.open();
      },
      openQRScaner() {
        let self = this;
        if (this.isNative) {
          cordova.exec(function (result) {
            let query = {
              to: result
            }
            if (result.indexOf('qr://transfer') == 0) {
              query = util.query2Obj(result.replace('qr://transfer', ''));
            }
            self.$router.push({
              path: self.link('/transfer',query)
            })
          }, null, 'QRCode', 'scan', []);
        }
      },
      linkBackup(account) {
        let query = {
          account: this.$route.query.account,
          from: this.$route.fullPath
        }
        return this.link('/wallet-backup-detail',query);
      },
      //TODO upload a qrcode image
      onFileUpload(e) {
        let file = this.$refs.qrfile;
      },
      openQRCodeModal() {
        this.$refs.qrcode.show();
      }
    },
    destroyed() {
      $.allowPanelOpen = true;
      $(this.$el).off('refresh');
//      $(this.$refs.content).off('scroll').off('touchmove').off('touchend');
      this.stopFetchingMarket = true;
    },
    mounted() {
      $.init();
      this.loadWallets();
      this.stopFetchingMarket = false;
      this.loadMarketInfo();
      if (this.isNative) {
        window.webview = {
          reload: () => {
            this.loadWallets();
          }
        }
        cordova.exec(null, null, 'statusBar', 'styleLightContent', []);
      }
      $(this.$el).on('refresh', '.pull-to-refresh-content', (e) => {
        this.loadWallets();
      })


      $(this.$refs.content).on('scroll', () => {
        let offset = $(this.$el).find('.row-top').offset().top;
        if (offset < -20) {
          let percent = Math.min((-40 - offset) / 40, 1);
          $(this.$refs.bar).css({background: `rgba(37, 40, 113, ${percent})`});
        }
        else {
          $(this.$refs.bar).css({background: 'rgba(37, 40, 113, 0)'});
        }
      }).on('touchmove', (e) => {
        $(this.$refs.bg).css({
          height: Math.max($(this.$refs.top).height() + $(this.$refs.top).offset().top, $(this.$refs.top).height()),
          transition: 'none',
          webkitTransition: 'none'
        })
        return true;
      }).on('touchend', () => {
        const end = () => {
          $(this.$refs.bg).attr('style', '');
          clearTimeout(timer);
        }
        let timer = setTimeout(end, 1000);
      });
      this.swiper.update();
    },

    computed: {
      qrcode() {
        return `qr://transfer?account=${this.currentWallet.account}`;
      },
      currentWallet() {
        if (this.wallets.length > 0) {
          return this.wallets[this.currentWalletIndex];
        }
        return {};
      },
      price() {
        if (!this.exchanges || this.exchanges.length == 0) {
          return '-';
        }
        else {
          let best_price = 0;
          this.exchanges.forEach((exchange) => {
            if (this.$i18n.locale == 'zh-CN') {
              best_price = Math.max(Number(exchange.price_rmb), best_price);
            }
            else {
              best_price = Math.max(Number(exchange.price_dollar), best_price);
            }
          });
          return best_price;
        }
      },
      swiper() {
        return this.$refs.mySwiper.swiper
      }
    },
    components: {
      swiper,
      swiperSlide,
      AccountImage,
      LeftPanel,
      AccountQRCode
    }
  }
</script>
<style lang="scss" scoped="">
  @import "../assets/styles/components/refresh_layer";

  /*::-webkit-scrollbar {*/
  /*display: none;*/
  /*}*/

  #bg {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    background: linear-gradient(to bottom, #150838 0%, #3853bc 49%, #18bce2 100%);
    height: 16.75rem;
    transition: height .2s;
  }

  #page-wallet-index {
    .content {
      top: 0;
      -webkit-overflow-scrolling: auto;
    }
  }

  .bar-nav.buttons-fixed .icon {
    font-size: 1.2rem;
    color: #fff;
  }

  .row-top {
    background: transparent; //linear-gradient(to bottom, #150838 0%, #3853bc 49%, #18bce2 100%);
    color: #fff;
    min-height: 14.8rem;
    display: flex;
    position: relative;
    align-items: center;
    justify-content: center;
    .loading {
      position: absolute;
    }
    &.ios {
      padding-top: 5px;
    }
    .icon {
      font-size: 1.2rem;
      color: #fff;
    }
    .swiper-container {
      width: 100%;
      display: block;
    }
  }

  .wallet-info {
    margin-top: 3.5rem;
    .account-header {
      display: inline-block;
    }
    .link-qrcode {
      color: #fff;
      .icon {
        position: relative;
        top: -1px;
        font-size: .8rem;
      }
    }
    .bak-tip{
      margin: .4rem 0;
    }
    .btn-backup {
      visibility: hidden;
      color: #fff;
      .red-dot {
        position: relative;
        display: inline-block;
        top: -8px;
        border-radius: 2px;
        width: 4px;
        height: 4px;
        background: #ff6e35;
      }
      .icon-right {
        position: relative;
        top: -2px;
        font-size: .6rem;
        margin-left:.2rem;
      }
    }

    .row-balance {
      display: flex;
      flex-direction: row;
      padding: 0 .75rem;
      .item {
        flex: 2;
        display: flex;
        flex-direction: column;
        align-items: flex-start;
        .digit {
          font-family: bebas;
          font-size: 1.1rem;
        }
      }
      .item-middle {
        flex: 1;
        align-items: center;
      }
      .value {
        align-items: flex-end;
      }
    }
  }

  .row-operation {
    height: 3.6rem;
    display: flex;
    padding: 0.75rem;
    background: #fff;
    position: relative;
    z-index: 1;
    .item {
      display: flex;
      flex: 2;
      font-weight: bold;
      flex-direction: row;
      justify-content: center;
      align-items: center;
      .gxicon {
        width: 35px;
        margin-right: 1.25rem;
      }
    }

    .sep {
      display: flex;
      height: 100%;
      width: 1px;
      background: #eee;
    }
  }

  .table-exchanges {
    position: relative;
    height: 25rem;
    background: #fff;
    .list-block {
      border-top: .2rem solid #e7e7e7;
      margin-top: 0;
      margin-bottom: 0;
    }

    .item-exchange .item-inner {
      display: flex;
      flex-direction: row;
      padding-top: .75rem;
      padding-bottom: .65rem;
      .title-ex-name {
        text-align: left;
        .asset-name{
          color: #3d3d3b;
        }
        .ex-name {
          font-size: .7rem;
          color: #888;
        }
      }

      .price {
        text-align: right;
        .price_secondary {
          font-size: .7rem;
          color: #888;
        }
      }

      .quote {
        height: 1.2rem;
        line-height: 1.2rem;
        width: 3.5rem;
        text-align: right;
        padding: 0 .2rem;
        font-weight: bold;
        font-size: .65rem;
        color: #fff;
        background: #d63932;
        &.green {
          background: #076f8a;
        }
      }
    }
  }

  .pull-to-refresh-layer {
    position: absolute;
    top: 1rem;
    .line-scale > div {
      background-color: #fff !important;
    }
  }

  .link-loyalty-program {
    z-index: 2;
    position: absolute;
    top: 5.5rem;
    right: 0;
    width: 4.1rem;
    height: 1.6rem;
    img{
      width: 100%;
      height: 100%;
    }
  }

</style>
