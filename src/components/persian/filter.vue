<template>
    <div class="filterable">

        <div class="panel">
            <div class="filter-item" v-for="(f , i) in filterCandidates" dir="rtl">
                <div class="filter-column">
                    <div class="form-group">
                        <select class="vue-form-control" v-for="group in filterGroups"
                                @input="selectColumn(f, i , $event)">
                            <option>انتخاب کنید</option>
                            <option v-for="x in group.filters" :value="JSON.stringify(x)"
                                    :selected="f.column && x.name === f.column.name">
                                {{x.title}}
                            </option>
                        </select>
                    </div>
                </div>
                <div class="filter-operator" v-if="f.column">
                    <div class="form-group">
                        <select class="vue-form-control" @input="selectOperator(f, i , $event)">
                            <option v-for="y in fetchOperators(f)" :value="JSON.stringify(y)"
                                    :selected="f.operator && y.name === f.operator.name">
                                {{y.title}}
                            </option>
                        </select>
                    </div>
                </div>
                <template v-if="f.column && f.operator">
                    <div class="filter-full" v-if="f.operator.component==='single'">
                        <input class="vue-form-control" v-model="f.query_1"/>
                    </div>

                    <template v-if="f.operator.component==='double'">
                        <div class="filter-query_1">
                            <input class="vue-form-control" v-model="f.query_1"/>
                        </div>

                        <div class="filter-query_2">
                            <input class="vue-form-control" v-model="f.query_2"/>
                        </div>
                    </template>

                    <template v-if="f.operator.component==='datetime_1'">
                        <div class="filter-query_1">
                            <input class="vue-form-control" v-model="f.query_1"/>
                        </div>
                        <div class="filter-query_2">
                            <select class="vue-form-control" v-model="f.query_2">
                                <option value="hours">ساعت</option>
                                <option value="days">روز</option>
                                <option value="months">ماه</option>
                                <option value="years">سال</option>

                            </select>
                        </div>

                    </template>

                    <template v-if="f.operator.component==='datetime_2'">

                        <div class="filter-query_2">
                            <select class="vue-form-control" v-model="f.query_1">
                                <option value="yesterday">دیروز</option>
                                <option value="today">امروز</option>
                                <option value="tomorrow">فردا</option>
                                <option value="last_month">ماه گذشته</option>
                                <option value="this_month">ماه جاری</option>
                                <option value="next_month">ماه آینده</option>
                                <option value="last_year">سال گذشته</option>
                                <option value="this_year">سال جاری</option>
                                <option value="next_year">سال آینده</option>

                            </select>
                        </div>

                    </template>


                </template>
                <div class="filter-remove" v-if="f">
                    <button @click="removeFilter(f , i)">X</button>
                </div>
            </div>

            <div class="filterable-export">
                <div>
                    <button class="vue-btn vue-btn-success" @click="addFilter">+</button>
                    <button class="vue-btn vue-btn-danger" @click="resetFilter"
                            v-if="this.appliedFilters.length>0">حذف فیلترها
                    </button>
                    <button class="vue-btn vue-btn-primary" @click="applyFilter">اعمال فیلتر</button>
                </div>
                <div>
                    <select dir="rtl" :disabled="loading" @input="updateOrderColumn">
                        <option v-for="column in orderables" :value="column.name" :selected="column && column.name=== query.order_column">
                            {{column.title}}
                        </option>
                    </select>
                    <strong class="vue-directions" @click="updateOrderDirection">
                        <span v-if="query.order_direction === 'asc'">&uarr;</span>
                        <span v-else>&darr;</span>
                    </strong>
                </div>
            </div>
            <div class="panel-body">
                <table class="vue-table" dir="rtl">
                    <slot name="thead"></slot>
                    <tbody>
                    <slot v-if="collection.data && collection.data.length" v-for="item in collection.data"
                          :item="item"></slot>
                    </tbody>
                </table>
            </div>

            <div class="panel-footer">
                <div>
                    <select dir="rtl" v-model="query.limit" @change="updateLimit" :disabled="loading">
                        <option value="10">١۰</option>
                        <option value="15">١۵</option>
                        <option value="25">۲۵</option>
                        <option value="50">۵۰</option>
                    </select>
                    <small> نمایش {{collection.from}} - {{collection.to}} از {{collection.total}} داده</small>
                </div>
                <div>
                    <button class="vue-btn vue-btn-secondary" :disabled="!collection.prev_page_url || loading" @click="prevPage">
                        قبلی
                    </button>
                    <button class="vue-btn  vue-btn-secondary" :disabled="!collection.next_page_url || loading" @click="nextPage">
                        بعدی
                    </button>
                </div>
            </div>
        </div>
    </div>


</template>


<script type="text/javascript">

    import Vue from 'vue'
    import axios from 'axios'


    export default {

        props: {
            url: String,
            order_by: String,
            filterGroups: Array,
            orderables: Array
        },
        data() {
            return {
                show: false,
                loading: true,
                appliedFilters: [],
                filterCandidates: [],

                query: {
                    order_column: this.order_by,
                    order_direction: 'desc',
                    filter_match: 'and',
                    limit: 10,
                    page: 1
                },
                collection: {
                    data: [],
                }

            }

        },

        computed: {

            fetchOperators() {
                return (f) => {
                    return this.availableOperators().filter((operator) => {

                        if (f.column && operator.parent.includes(f.column.type)) {
                            return operator;
                        }

                    });
                }
            },
        },
        mounted() {
            this.fetch();
        },
        methods: {

            updateOrderDirection() {

                if (this.query.order_direction === 'desc') {
                    this.query.order_direction = 'asc'
                } else {
                    this.query.order_direction = 'desc'

                }
                this.applyChange();

            },
            updateOrderColumn(e) {
                const value = e.target.value;
                Vue.set(this.query, 'order_column', value);
                this.applyChange();
            },


            resetFilter() {
                this.show = false;
                this.appliedFilters.splice(0);
                this.filterCandidates.splice(0);
                this.query.page = 1;
                this.applyChange();

            },
            applyFilter() {
                Vue.set(this.$data, 'appliedFilters',
                    JSON.parse(JSON.stringify(this.filterCandidates))
                );
                this.query.page = 1;
                this.applyChange();

            },

            removeFilter(f, i) {
                this.filterCandidates.splice(i, 1);
            },


            selectOperator(f, i, e) {

                let value = e.target.value;

                if (value.length === 0) {
                    Vue.set(this.filterCandidates[i], 'operator', value);
                    return
                }


                let obj = JSON.parse(value);

                Vue.set(this.filterCandidates[i], 'operator', obj)

                this.filterCandidates[i].query_1 = null;
                this.filterCandidates[i].query_2 = null;


                //set defult query

                switch (obj.name) {
                    case  'in_the_period':
                        this.filterCandidates[i].query_1 = 'today';
                        break;

                    case 'in_the_past':
                    case 'in_the_next':
                        this.filterCandidates[i].query_1 = 28;
                        this.filterCandidates[i].query_2 = 'days';
                        break;

                }


            },
            selectColumn(f, i, e) {
                let value = e.target.value;
                if (value.length === 0) {
                    Vue.set(this.filterCandidates[i], 'column', value);
                    return
                }

                let obj = JSON.parse(value);

                Vue.set(this.filterCandidates[i], 'column', obj)


                switch (obj.type) {
                    case 'numeric':
                        this.filterCandidates[i].operator = this.availableOperators()[4];
                        this.filterCandidates[i].query_1 = null;
                        this.filterCandidates[i].query_2 = null;
                        break;
                    case 'string':
                        this.filterCandidates[i].operator = this.availableOperators()[6];
                        this.filterCandidates[i].query_1 = null;
                        this.filterCandidates[i].query_2 = null;
                        break;

                    case 'datetime':
                        this.filterCandidates[i].operator = this.availableOperators()[9];
                        this.filterCandidates[i].query_1 = 28;
                        this.filterCandidates[i].query_2 = 'days';
                        break;


                    case 'counter':
                        this.filterCandidates[i].operator = this.availableOperators()[14];
                        this.filterCandidates[i].query_1 = null;
                        this.filterCandidates[i].query_2 = null;
                        break;
                }


            },

            addFilter() {
                this.show = true;
                this.filterCandidates.push({
                    column: '',
                    operator: '',
                    query_1: null,
                    query_2: null
                });

            },
            applyChange() {
                this.fetch();
            },
            updateLimit() {
                this.query.page = 1;
                this.applyChange()
            },
            prevPage() {
                if (this.collection.prev_page_url) {
                    this.query.page = Number(this.query.page) - 1;
                    this.applyChange();
                }
            },

            nextPage() {
                if (this.collection.next_page_url) {
                    this.query.page = Number(this.query.page) + 1;
                    this.applyChange();
                }
            },

            getFilters() {
                const f = {};
                this.appliedFilters.forEach((filter, i) => {


                    f[`f[${i}][column]`] = filter.column.name;
                    f[`f[${i}][operator]`] = filter.operator.name;
                    f[`f[${i}][query_1]`] = filter.query_1;
                    f[`f[${i}][query_2]`] = filter.query_2;


                });
                return f;
            },
            fetch() {

                this.loading = true;
                const filters = this.getFilters();
                const params = {
                    ...filters,
                    ...this.query
                };
                axios.get(this.url, {params: params})
                    .then((res) => {
                        if (res.status == 200) {
                            Vue.set(this.$data, 'collection', res.data.collection);
                            this.query.page = res.data.collection.current_page
                        }

                    }).catch((error) => {
                    alert('error!!')
                })
                    .finally(() => {
                        this.loading = false
                    })
            },
            availableOperators() {
                return [
                    {title: 'برابر باشد با', name: 'equal_to', parent: ['numeric', 'string'], component: 'single'},
                    {
                        title: 'برابر نباشد با',
                        name: 'not_equal_to',
                        parent: ['numeric', 'string'],
                        component: 'single'
                    },
                    {title: 'کوچکتر از', name: 'less_than', parent: ['numeric'], component: 'single'},
                    {title: 'بزرگتر از', name: 'greater_than', parent: ['numeric'], component: 'single'},
                    {title: 'بین', name: 'between', parent: ['numeric'], component: 'double'},
                    {title: 'نباشد بین', name: 'not_between', parent: ['numeric'], component: 'double'},

                    {title: 'شامل', name: 'contains', parent: ['string'], component: 'single'},
                    {title: 'شروع شود با', name: 'starts_with', parent: ['string'], component: 'single'},
                    {title: 'تمام شود با', name: 'ends_with', parent: ['string'], component: 'single'},

                    {title: 'گذشته', name: 'in_the_past', parent: ['datetime'], component: 'datetime_1'},
                    {title: 'آینده', name: 'in_the_next', parent: ['datetime'], component: 'datetime_1'},
                    {title: 'دوره', name: 'in_the_period', parent: ['datetime'], component: 'datetime_2'},


                    {title: 'برابر باشد با', name: 'equal_to_count', parent: ['counter'], component: 'single'},
                    {title: 'برابر نباشد با', name: 'not_equal_to_count', parent: ['counter'], component: 'single'},
                    {title: 'کوچکتر از', name: 'less_than_count', parent: ['counter'], component: 'single'},
                    {title: 'بزرگتر از', name: 'greater_than_count', parent: ['counter'], component: 'single'},


                ]
            }
        }

    }
</script>

<style scoped>
    @import "/css/VueFilter/VueFilter.css";
</style>

