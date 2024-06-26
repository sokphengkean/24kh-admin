<script setup lang="ts">
import type { ICategory, ICategoryQuery } from '~/types/Category';
import CreateOrUpdate from "./_createOrUpdate/index.vue"
import { createColumnHelper, FlexRender, getCoreRowModel, getFilteredRowModel, getPaginationRowModel, getSortedRowModel, useVueTable } from '@tanstack/vue-table'
import type { ColumnDef, Table } from '@tanstack/vue-table';
import Checkbox from '~/components/ui/checkbox/Checkbox.vue';
import DropDown from './_table/DropDown.vue';
import { LoaderCircle, MoreHorizontal } from 'lucide-vue-next';
import { Button } from '~/components/ui/button';
import { useToast } from '~/components/ui/toast';

definePageMeta({
    breadcrumb: [
        {
            name: 'Categories',
            path: '/categories',
        },
    ],
})

const category = useCategory();

const categoryList = ref<ICategory[]>([]);
const filterCategories = ref<ICategory[]>([]);
const selectedCategory = ref<ICategory[]>([]);
const selectedOne = ref<ICategory>();

const dialogModal = ref<boolean>(false);

const loading = ref<boolean>(true);

const { toast } = useToast();

const modalDeleteSelectedList = defineModel<boolean>();

onBeforeMount(async () => {
    const resp = await category.getAll(queryCategories.value);
    if (resp.statusCode === 200) {
        categoryList.value = resp.data;
        filterCategories.value = resp.data;
    }
    loading.value = false;
});

watchEffect(() => {
    filterCategories.value = categoryList.value.filter((category) => {
        if (!queryCategories.value.keyword) return true;
        return category.nameEn.toLowerCase().includes(queryCategories.value.keyword.toLowerCase());
    });
});

const queryCategories = ref<ICategoryQuery>({
    page: 1,
    limit: 100,
    keyword: '',
    sortBy: 'nameEn',
    orderBy: 'asc'
})

const columns: ColumnDef<ICategory>[] = [
    {
        id: 'select',
        header: ({ table }) => h(Checkbox, {
            'checked': table.getIsAllPageRowsSelected() || (table.getIsSomePageRowsSelected() && 'indeterminate'),
            'onUpdate:checked': value => table.toggleAllPageRowsSelected(!!value),
            'ariaLabel': 'Select all',
        }),
        cell: ({ row }) => h(Checkbox, {
            'checked': row.getIsSelected(),
            'onUpdate:checked': value => row.toggleSelected(!!value),
            'ariaLabel': 'Select row',
        }),
        // enableSorting: false,
        // enableHiding: false,
    },
    {
        accessorKey: 'nameEn',
        header: 'Name EN',
        cell: ({ row }) => h('div', { class: 'capitalize' }, row.getValue('nameEn'))
    },
    {
        accessorKey: 'nameKh',
        header: 'Name KH',
        // header: ({ column }) => {
        //     return h(Button, {
        //         variant: 'ghost',
        //         onClick: () => column.toggleSorting(column.getIsSorted() === 'asc'),
        //     }, () => ['Email', h(ArrowUpDown, { class: 'ml-2 h-4 w-4' })])
        // },
        cell: ({ row }) => h('div', { class: 'capitalize' }, row.getValue('nameKh')),
    },
    {
        id: 'actions',
        header: ({ column }) => {
            if (table.getFilteredSelectedRowModel().rows.length > 1) {
                selectedCategory.value = table.getFilteredSelectedRowModel().rows.map(row => row.original);
                return h('div', { class: 'relative text-right' }, h(Button, {
                    variant: 'destructive',
                    size: 'xs',
                    onClick: () => modalDeleteSelectedList.value = true,
                }, 'Delete Selected'))
            }
        },
        cell: ({ row }) => {
            const category = row.original

            return h('div', { class: 'relative text-right' }, h(DropDown, {
                category,
                onDeleted: () => deleteOne(category),
                onUpdated: (category: ICategory) => {
                    selectedOne.value = category;
                    dialogModal.value = true;
                }
            }))
        },
    },
]

const deleteOne = async (item: ICategory) => {
    const resp = await category.remove(item.id);
    if (resp.statusCode === 200) {
        categoryList.value = categoryList.value.filter((category) => category.id !== item.id);
        filterCategories.value = filterCategories.value.filter((category) => category.id !== item.id);
        toast({
            title: 'Success',
            description: resp.message,
            variant: 'default',
            duration: 5000,
        });
    } else {
        toast({
            title: 'Error',
            description: resp.message,
            variant: 'destructive',
            duration: 5000,
        });
    }
}

const onDeleteSelected = () => {
    // TODO: Delete selected items
    filterCategories.value = filterCategories.value.filter((category) => !selectedCategory.value.includes(category));
    selectedCategory.value = [];
    modalDeleteSelectedList.value = false;
    table.toggleAllRowsSelected(false);
}

const table = useVueTable({
    get data() { return filterCategories.value },
    get columns() { return columns },
    getCoreRowModel: getCoreRowModel(),
    getPaginationRowModel: getPaginationRowModel({ initialSync: true, }),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    initialState: {
        pagination: {
            pageSize: 10,
            pageIndex: 0,
        },
    },
    autoResetPageIndex: false,
})
const onCategorySuccess = (category: ICategory) => {
    if (selectedOne.value) {
        const index = categoryList.value.findIndex((item) => item.id === category.id);
        categoryList.value[index] = category;
        selectedOne.value = undefined;
    } else {
        categoryList.value = [category, ...categoryList.value];
    }
}

watchEffect(() => {
    // get the selected rows
    selectedCategory.value = table.getFilteredSelectedRowModel().rows.map(row => row.original);
})

watchEffect(() => {
    if (!dialogModal.value) {
        selectedOne.value = undefined;
    }
})

</script>

<template>
    <div class="flex gap-2 justify-between items-center pb-4">
        <UIInput class="max-w-sm" placeholder="Filter name..."
            :model-value="table.getColumn('nameEn')?.getFilterValue() as string"
            @update:model-value=" table.getColumn('nameEn')?.setFilterValue($event)" />
        <CreateOrUpdate @on-success="onCategorySuccess" />
    </div>
    <transition-fade tag="div" appear mode="out-in" class="rounded-md border overflow-auto h-full">
        <UITable v-if="!loading">
            <UITableHeader>
                <UITableRow v-for="headerGroup in table.getHeaderGroups()" :key="headerGroup.id">
                    <UITableHead v-for="header in headerGroup.headers" :key="header.id">
                        <FlexRender v-if="!header.isPlaceholder" :render="header.column.columnDef.header"
                            :props="header.getContext()" />
                    </UITableHead>
                </UITableRow>
            </UITableHeader>
            <UITableBody>
                <template v-if="table.getRowModel().rows?.length">
                    <UITableRow v-for="row in table.getRowModel().rows" :key="row.id"
                        :data-state="row.getIsSelected() && 'selected'">
                        <UITableCell v-for="cell in row.getVisibleCells()" :key="cell.id">
                            <FlexRender :render="cell.column.columnDef.cell" :props="cell.getContext()" />
                        </UITableCell>
                    </UITableRow>
                </template>

                <UITableRow v-else>
                    <UITableCell :colspan="columns.length" class="h-24 text-center">
                        No results.
                    </UITableCell>
                </UITableRow>
            </UITableBody>
        </UITable>
        <div v-else class="flex items-center justify-center grow h-full">
            <LoaderCircle class="animate-spin mr-4" /> Loading...
        </div>
    </transition-fade>
    <div class="mt-6">
        <DataTablePagination :table="table" />
    </div>
    <CreateOrUpdate v-if="selectedOne" @on-success="onCategorySuccess" :item="selectedOne" v-model="dialogModal" />
    <ModalConfirmDelete v-model="modalDeleteSelectedList" title="Delete Selected Categories"
        @on-confirm="onDeleteSelected" description="Are you about to delete these selected items?" variant="danger" />
</template>