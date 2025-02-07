<script setup>
import { ref, computed, onMounted } from 'vue';
import { deleteHourWorked, getHourWorkedByDate, updateHourWorked } from '../../service/HourService';
import { DateTime } from 'luxon';
import { getCompanies } from '../../service/CompanyService';
import { watch } from 'vue';
import { useToast } from 'primevue/usetoast';

const toast = useToast();

// Campos para la búsqueda de horas trabajadas
let startDate = ref(null);
let formattedStartDate = "";
let hoursWorkedArray = ref([]);
const visibleTable = ref(false);
const visibleMsg = ref(false);
const loading = ref(false);

let hour = ref({});

let editDialog = ref(false);
let deleteDialog = ref(false);

// Campos para la edición
let company = ref(null);
let companies = ref(null);
let editDateWork = ref(null);
let editDayWork = ref("");
let editStartTime = ref(null);
let editEndTime = ref(null);
let editTotalHours = ref(null);

const searchDates = async () => {
    loading.value = true;

    let dateStart = DateTime.fromJSDate(startDate.value, { zone: 'America/Bogota' });
    formattedStartDate = dateStart.toFormat('yyyy-MM-dd');

    let data = {
        start_date: formattedStartDate
    }

    const response = await getHourWorkedByDate(data);
    if (response.length > 0) {
        hoursWorkedArray.value = response;
        visibleTable.value = true;
        visibleMsg.value = false;
    } else {
        visibleTable.value = false;
        visibleMsg.value = true;
    }

    loading.value = false;
}

const isFormValidSearchDates = computed(() => {
    return startDate.value;
});

const isValidHoursWorked = computed(() => {
    return hoursWorkedArray.value && hoursWorkedArray.value.length > 0;
});

const getDayOfWeek = () => {
    if (editDateWork.value) {
        const date = DateTime.fromJSDate(editDateWork.value, { zone: 'America/Bogota' });
        const daysOfWeek = ['domingo', 'lunes', 'martes', 'miércoles', 'jueves', 'viernes', 'sábado'];
        const dayIndex = date.weekday % 7;
        editDayWork.value = daysOfWeek[dayIndex];
    } else {
        editDayWork.value = null
    }

};

const editHour = (dataHour) => {
    hour.value = { ...dataHour };

    editDateWork.value = DateTime.fromISO(dataHour.date_worked).toJSDate();
    editDayWork.value = dataHour.day_worked;
    editStartTime.value = DateTime.fromFormat(dataHour.start_time, 'h:mma').toJSDate();
    editEndTime.value = DateTime.fromFormat(dataHour.end_time, 'h:mma').toJSDate();
    editTotalHours.value = parseFloat(dataHour.total_hours);
    company.value = companies.value.find(c => c.name === dataHour.name_company);

    editDialog.value = true;
}

const updateHour = async () => {

    const formattedStartTime = DateTime.fromJSDate(editStartTime.value, { zone: 'America/Bogota' }).toFormat('h:mma');
    const formattedEndTime = DateTime.fromJSDate(editEndTime.value, { zone: 'America/Bogota' }).toFormat('h:mma');
    const date = DateTime.fromJSDate(editDateWork.value, { zone: 'America/Bogota' });
    let data = {
        "day_worked": editDayWork.value,
        "date_worked": date.toFormat('yyyy-MM-dd'),
        "start_time": formattedStartTime,
        "end_time": formattedEndTime,
        "id_company": +company.value.code,
        "total_hours": editTotalHours.value.toString()
    }

    try {
        await updateHourWorked(hour.value.id_hour_worked, data)
        toast.add({ severity: 'success', summary: 'Successful', detail: `Hora editada con éxito`, life: 3000 });
    } catch (error) {
        await searchDates()
        toast.add({ severity: 'warning', summary: 'Error', detail: `Error al editar`, life: 3000 });
    }

    editDialog.value = false
}

const deleteHour = async () => {
    try {
        await deleteHourWorked(hour.value.id_hour_worked)
        toast.add({ severity: 'success', summary: 'Successful', detail: `Hora eliminada con éxito`, life: 3000 });
    } catch (error) {
        toast.add({ severity: 'warning', summary: 'Error', detail: `Error al borrar el registro.`, life: 3000 });
    }

    deleteDialog.value = false
    startDate.value = null
    visibleTable.value = false

}

const confirmDeleteHour = (dataHour) => {
    hour.value = dataHour;
    deleteDialog.value = true
    console.log(dataHour);
}

watch(editDateWork, getDayOfWeek)

onMounted(async () => {
    companies.value = await getCompanies();
});

</script>


<template>
    <div class="grid">
        <div class="col-12">
            <div class="card">
                <h5>Horas registradas en el sistema.</h5>
                <div class="p-fluid formgrid grid">
                    <div class="field col-12 md:col-5">
                        <label for="firstname2">Fecha de busqueda</label>
                        <Calendar :showIcon="true" dateFormat="dd/mm/yy" :showButtonBar="true" v-model="startDate"
                            placeholder="Fecha inicio" />
                    </div>

                    <div class="field col-12 md:col-2 md:mt-4">
                        <Button label="Buscar" :disabled="!isFormValidSearchDates" :loading="loading"
                            @click="searchDates" icon="pi pi-search" />
                    </div>

                    <h4 class="col-12 text-center" v-if="visibleMsg">No hay horas registradas.</h4>
                    <div v-if="visibleTable" class="field col-12">
                        <DataTable :value="hoursWorkedArray" size="small" tableStyle="min-width: 10rem">
                            <Column header="Acciones" headerStyle="min-width:5rem;">
                                <template #body="slotProps">
                                    <Button icon="pi pi-pencil" class="mr-2" severity="success" rounded
                                        @click="editHour(slotProps.data)" />
                                    <Button icon="pi pi-trash" class="mt-2" severity="danger" rounded
                                        @click="confirmDeleteHour(slotProps.data)" />
                                </template>
                            </Column>
                            <Column field="date_worked" header="Fecha"></Column>
                            <Column field="name_company" header="Empresa"></Column>
                            <Column field="horario" header="Horario"></Column>
                            <Column field="total_hours" header="Horas"></Column>
                            <Column field="valor_total" header="Valor total"></Column>
                        </DataTable>

                        <Dialog v-model:visible="editDialog" :style="{ width: '450px' }" header="Hora registrada"
                            :modal="true" class="p-fluid">

                            <!-- Fecha trabajada -->
                            <Calendar v-model="editDateWork" :showIcon="true" dateFormat="dd/mm/yy"
                                :showButtonBar="true" placeholder="Fecha trabajada" />

                            <!-- Día trabajado -->
                            <FloatLabel class="mt-4">
                                <InputText v-model="editDayWork" type="text" disabled />
                            </FloatLabel>

                            <!-- Horas de trabajo -->
                            <h6>Horas de trabajo</h6>
                            <div class="grid formgrid">
                                <div class="col-12 mb-4 lg:col-6 lg:mb-0">
                                    <IconField iconPosition="right">
                                        <Calendar v-model="editStartTime" hourFormat="12" :showTime="true"
                                            :timeOnly="true" :showSeconds="false" :showOnFocus="true"
                                            placeholder="Hora Inicio" />
                                        <InputIcon class="pi pi-calendar-times" />
                                    </IconField>
                                </div>
                                <div class="col-12 mb-2 lg:col-6 lg:mb-0">
                                    <IconField iconPosition="right">
                                        <Calendar v-model="editEndTime" hourFormat="12" :showTime="true"
                                            :timeOnly="true" :showSeconds="false" :showOnFocus="true"
                                            placeholder="Hora Fin" />
                                        <InputIcon class="pi pi-calendar-times" />
                                    </IconField>
                                </div>
                            </div>

                            <!-- Total Horas -->
                            <h5 class="mb-4">Total Horas</h5>
                            <FloatLabel>
                                <InputNumber v-model="editTotalHours" inputId="horizontal-buttons" showButtons
                                    buttonLayout="horizontal" :step="0.50" mode="decimal" :min="0" :max="24" fluid>
                                    <template #incrementbuttonicon>
                                        <span class="pi pi-plus" />
                                    </template>
                                    <template #decrementbuttonicon>
                                        <span class="pi pi-minus" />
                                    </template>
                                </InputNumber>
                            </FloatLabel>

                            <!-- Empresa -->
                            <h5>Escoga la empresa</h5>
                            <Dropdown v-model="company" :options="companies" optionLabel="name"
                                placeholder="Seleccione una empresa" />

                            <template #footer>
                                <Button label="Cancelar" icon="pi pi-times" text="" @click="editDialog = false" />
                                <Button label="Editar" icon="pi pi-check" text="" @click="updateHour" />
                            </template>
                        </Dialog>

                        <Dialog v-model:visible="deleteDialog" :style="{ width: '450px' }" header="Confirmar"
                            :modal="true">
                            <div class="flex align-items-center justify-content-center">
                                <i class="pi pi-exclamation-triangle mr-3" style="font-size: 2rem" />
                                <span v-if="hour">Seguro que quieres borrar el registro del <b>{{ hour.date_worked
                                        }}</b>?</span>
                            </div>
                            <template #footer>
                                <Button label="No" icon="pi pi-times" text @click="deleteDialog = false" />
                                <Button label="Si" icon="pi pi-check" text @click="deleteHour" />
                            </template>
                        </Dialog>

                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
