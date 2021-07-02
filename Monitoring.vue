<template>
    <div class="monitoring">
        <div class="monitoring__block">
            <h2 class="title">Мониторинг</h2>
            <Table class="monitoring__table"
                   :head="['Микросервис оборудования','Дата и время изменения статуса','Статус','']">
                <tr class="table__row" v-for="item in items" :key="item.sid">
                    <td class="table__col">{{item.name}}</td>
                    <td class="table__col">{{item.state_date}}</td>
                    <td class="table__col">
                    <span class="monitoring__status">
                       <span class="monitoring__status-text">{{item.state_name}}</span>
                        <button class="monitoring__status-icon reset-btn" @click="showInfoModal(item.state_message)">
                            <img src="/images/info.svg" alt="" width="15" height="15">
                        </button>
                    </span>
                    </td>
                    <td class="table__col">
                        <a :href="'/monitor/'+item.sid+'/download-log'" class="monitoring__download-log reset-btn">
                            Скачать последний лог
                        </a>
                    </td>
                </tr>
            </Table>
        </div>
        <div class="monitoring__block">
            <h2 class="title">Журнал логов</h2>
            <form class="monitoring__form" @submit.prevent="getJournal">
                <Select name="sid" :options="selectItems" defaultValue="0"/>
                <div class="monitoring__dates">
                    <input name="sdate" type="datetime-local" class="input" :max="$store.getters.getTime">
                    &nbsp;—&nbsp;
                    <input name="edate" type="datetime-local" class="input" :max="$store.getters.getTime">
                </div>
                <div class="monitoring__limit">
                    <input name="limit" type="number" class="input" placeholder="Лимит" id="limit">
                </div>
                <Button type="submit" class="btn_primary btn_small">Загрузить</Button>
            </form>
            <Table class="monitoring__table"
                   :head="['Микросервис оборудования','Дата и время создания файла','']">
                <tr class="table__row" v-for="item in journal" :key="item.sid">
                    <td class="table__col">{{item.name}}</td>
                    <td class="table__col">{{item.created}}</td>
                    <td class="table__col">
                        <a class="table__link" :href="'/monitor/download-log?path='+item.file_path">Скачать файл</a>
                    </td>
                </tr>
            </Table>
        </div>
        <Modal class="modal_info" @close="showModal=false" v-if="showModal">
            <button class="modal__close reset-btn" type="button" @click="showModal=false">
                <img src="/images/close.svg" alt="" width="24" height="24">
            </button>
            <div class="modal__info">
                <img src="/images/info.svg" alt="" class="modal__info-icon" width="20" height="20">
                <div class="modal__info-text">Информация</div>
            </div>
            <div class="modal__body">{{info}}</div>
        </Modal>
    </div>
</template>

<script>
    import Cookie from 'js-cookie'

    import Table from "@/js/components/Table"
    import Select from "@/js/components/Select"
    import Button from "@/js/components/Button"
    import api from "@/js/api";

    export default {
        name: "Monitoring",
        components: {
            Table, Select, Button,
            Modal: () => import('@/js/components/Modal'),
        },
        data() {
            return {
                items: [],
                journal: [],
                showModal: false,
                info: null,
                isFirstRequest: true,
                monitorInterval: null
            }
        },
        computed: {
            selectItems() {

                let arr = this.items.map(item => {
                    return {name: item.name, value: item.sid}
                });

                arr.unshift({name: "Выберите микросервис", value: "0"});
                return arr;
            }
        },
        mounted() {
            this.getItems();
            this.monitorInterval = setInterval(() => {
                this.getItems()
            }, 1000 * 15);

            if (Cookie.get("no_log_file")) {
                alert("Файл отсутствует");
                Cookie.remove("no_log_file");
            }
        },
        destroyed() {
            clearInterval(this.monitorInterval)
        },
        methods: {
            async getItems() {
                if (this.isFirstRequest) this.$store.dispatch("setLoading", true);
                try {
                    const res = await api.monitor.index();
                    const response = res.data;
                    this.items = response.data;
                    this.isFirstRequest = false;
                } catch (error) {
                    this.$toastr.e(error.message, error.response.data.message);
                    clearInterval(this.monitorInterval);
                    Cookie.remove('token');
                    this.$router.push("/login").catch(error => {
                        console.info(error.message);
                    });
                }

                this.$store.dispatch("setLoading", false);
            },
            async getJournal(e) {
                this.$store.dispatch("setLoading", true);
                try {
                    let formData = new FormData(e.target);
                    if (formData.get("sid") === "0") {
                        formData.delete("sid")
                    }
                    const params = new URLSearchParams(formData).toString();
                    const res = await api.monitor.logs(params);
                    this.journal = res.data.data;
                } catch (error) {
                    this.$toastr.e(error.message, error.response.data.message);
                }
                this.$store.dispatch("setLoading", false);
            },
            showInfoModal(info) {
                this.showModal = true;
                this.info = info
            }
        }
    }
</script>

<style lang="scss">
    @import '~@/scss/variables.scss';

    .monitoring {
        &__block {
            &:not(:last-child) {
                margin-bottom: 40px;
            }
        }

        &__table {
            max-width: 920px;
            text-align: left;
        }

        &__status {
            display: flex;
            justify-content: space-between;
            align-items: center;
            min-width: 140px;

            &-icon {
                display: flex;
                padding: 5px;
                filter: invert(35%) sepia(45%) saturate(1613%) hue-rotate(185deg) brightness(103%) contrast(81%);

                &:hover {
                    filter: none;
                }
            }
        }

        &__download-log {
            white-space: nowrap;
            color: $primary;
            font-size: 14px;
            text-decoration: none;
        }

        &__form {
            display: flex;
            align-items: center;
            justify-content: space-between;
        }

        &__dates {
            display: flex;
            align-items: center;
        }

        &__limit {

            .input {
                width: 80px;
            }
        }
    }
</style>
