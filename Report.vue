<template>
	<div class="app-start-box" v-if="isAppLoading">
		<v-progress-circular indeterminate></v-progress-circular>
	</div>

	<div v-else class="report-container">
		<div class="filters-box">
			<div class="mode-switch">
				<v-select
					class="w-25"
					label="Режим отчёта"
					v-model="mode"
					:items="[
						{ title: 'По сотрудникам', value: 'byEmployee' },
						{ title: 'По задачам', value: 'byTask' },
					]"
				/>

				<v-select
					v-if="mode === 'byEmployee'"
					v-model="employeeFilter"
					:items="employeesSelect"
					label="Сотрудник"
					class="w-25"
					item-title="title"
					item-value="value"
					:searchable="true"
					:filterable="true"
					placeholder="Выберите сотрудника"
				/>

				<template v-if="mode === 'byTask'">
					<v-select
						v-model="taskFilters.status"
						:items="[
							{ title: 'Все', value: 'all' },
							{ title: 'В работе', value: 'in_progress' },
							{ title: 'Закрыта', value: 'completed' },
						]"
						label="Статус задачи"
						class="w-25"
					/>

					<v-date-input
						v-model="taskFilters.period.from"
						label="Дата закрытия (с)"
						variant="outlined"
						density="compact"
						class="w-25"
						autocomplete="off"
					clearable
					/>

					<v-date-input
						v-model="taskFilters.period.to"
						label="Дата закрытия (по)"
						variant="outlined"
						density="compact"
						class="w-25"
						clearable
						autocomplete="off"
					/>
				</template>
			</div>

			<v-btn
				density="default"
				color="primary"
				:loading="isLoading"
				:disabled="isLoading"
				@click="build"
			>
				Сформировать отчёт
			</v-btn>
		</div>

		<div class="report">
			<v-table
				:loading="isLoading"
				v-if="mode === 'byEmployee' && reportByEmployee.length"
				:hover="true"
				density="comfortable"
				height="350px"
				fixed-header
				fixed-footer
			>
				<thead>
					<tr>
						<th>Сотрудник</th>
						<th>Кол-во задач</th>
						<th>Запланировано</th>
						<th>Фактически</th>
						<th>Суммарная длительность задач (дн.)</th>
					</tr>
				</thead>
				<tbody>
					<tr
						style="cursor: pointer"
						v-for="row in reportByEmployee"
						:key="row.key"
						class="table-row"
					>
						<td>
							<v-tooltip location="top">
								<template #activator="{ props }">
									<span
										v-bind="props"
										style="cursor: pointer"
										@click="onVisitUser(row.id)"
									>
										{{ row.employee }}
									</span>
								</template>
								<span>Нажмите, чтобы открыть карточку сотрудника</span>
							</v-tooltip>
						</td>

						<td>{{ row.tasksCount }}</td>
						<td>{{ row.plannedFormatted }}</td>
						<td>{{ row.factFormatted }}</td>
						<td>{{ row.days }}</td>
					</tr>
				</tbody>
				<tfoot>
					<tr class="total-row">
						<td><b>Итого</b></td>
						<td>
							<b>{{ totalsByEmployee.tasks }}</b>
						</td>
						<td>
							<b>{{ totalsByEmployee.plannedFormatted }}</b>
						</td>
						<td>
							<b>{{ totalsByEmployee.factFormatted }}</b>
						</td>
						<td></td>
					</tr>
				</tfoot>
			</v-table>

			<v-table
				v-else-if="mode === 'byTask' && reportByTask.length"
				class="report-table"
				density="comfortable"
				height="350px"
				fixed-header
				fixed-footer
			>
				<thead>
					<tr>
						<th>Задача</th>
						<th>Сотрудник</th>
						<th>Запланировано</th>
						<th>Фактически</th>
						<th>Длительность задачи (дн.)</th>
					</tr>
				</thead>
				<tbody>
					<tr
						class="task-row"
						v-for="row in reportByTask"
						:key="row.key"
						style="cursor: pointer"
					>
						<td class="task-title">
							<v-tooltip location="top">
								<template #activator="{ props }">
									<span
										v-bind="props"
										style="cursor: pointer"
										@click="onVisitTask(row.employeeId, row.taskId)"
									>
										{{ row.taskTitle }}
									</span>
								</template>
								<span>Нажмите, чтобы открыть карточку задачи</span>
							</v-tooltip>
						</td>

						<td>{{ row.employee }}</td>
						<td>{{ row.plannedFormatted }}</td>
						<td
							:class="{ overdue: row.fact > row.planned }"
							style="position: relative; padding: 4px"
						>
							<v-tooltip top>
								<template #activator="{ props }">
									<div
										v-bind="props"
										style="
											position: relative;
											width: 100%;
											display: flex;
											align-items: center;
										"
									>
										<div
											:style="{
												position: 'absolute',
												top: 0,
												left: 0,
												height: '100%',
												width:
													Math.min((row.fact / (row.planned || 1)) * 100, 100) +
													'%',
												backgroundColor:
													row.fact > row.planned ? '#ffcccc' : '#d0f0fd',
												zIndex: 0,
												borderRadius: '4px',
												transition: 'width 0.3s ease',
											}"
										></div>
										<span
											style="position: relative; z-index: 1; padding: 0 4px"
										>
											{{ row.factFormatted }}
										</span>
									</div>
								</template>
								<span>
									{{
										row.fact > row.planned
											? `Превышение: ${formatDuration(row.fact - row.planned)}`
											: 'В пределах плана'
									}}
								</span>
							</v-tooltip>
						</td>
						<td>{{ row.days }}</td>
					</tr>
				</tbody>
				<tfoot>
					<tr class="total-row">
						<td>
							<div class="d-flex align-center gap-2">
								<b style="margin-right: 5px">Итого</b>
								<v-chip size="small" color="primary" variant="tonal">
									{{ totalsByTask.tasks }} задач
								</v-chip>
							</div>
						</td>
						<td></td>
						<td>
							<b>{{ totalsByTask.plannedFormatted }}</b>
						</td>
						<td>
							<b>{{ totalsByTask.factFormatted }}</b>
						</td>
						<td></td>
					</tr>
				</tfoot>
			</v-table>

			<v-alert v-if="!isReportBuilt" type="info" variant="tonal">
				Выберите фильтры и нажмите "Сформировать отчёт"
			</v-alert>

			<v-alert
				v-else-if="
					(mode === 'byEmployee' && !reportByEmployee.length) ||
					(mode === 'byTask' && !reportByTask.length)
				"
				type="warning"
				variant="tonal"
			>
				Нет данных для выбранных фильтров
			</v-alert>
		</div>
	</div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, watch } from 'vue'

function formatDuration(seconds: number): string {
	if (!seconds || seconds <= 0) return '0м'
	const h = Math.floor(seconds / 3600),
		m = Math.floor((seconds % 3600) / 60),
		s = seconds % 60
	const parts: string[] = []
	if (h) parts.push(`${h}ч`)
	if (m) parts.push(`${m}м`)
	if (!h && s) parts.push(`${s}с`)
	return parts.join(' ')
}

function getTaskDurationDays(task: Task): number {
	const startRaw = task.startDatePlan || task.createdDate
	if (!startRaw) return 0

	const start = new Date(startRaw)
	const end = task.closedDate ? new Date(task.closedDate) : new Date()

	const startDate = new Date(
		start.getFullYear(),
		start.getMonth(),
		start.getDate(),
	)
	const endDate = new Date(end.getFullYear(), end.getMonth(), end.getDate())

	const diffMs = endDate.getTime() - startDate.getTime()

	return Math.max(1, Math.ceil(diffMs / (1000 * 60 * 60 * 24)))
}

const onVisitUser = (id: string) => {
	BX24.openPath(`/company/personal/user/${id}/`, () => {})
}
const onVisitTask = (employeeId: string, taskId: string) => {
	console.log('click')
	BX24.openPath(
		`/company/personal/user/${employeeId}/tasks/task/view/${taskId}/`,
		() => {},
	)
}

const sleep = (ms: number) => new Promise(res => setTimeout(res, ms))

interface Task {
	id: number
	title: string
	responsibleId: number
	timeEstimate: number
	timeSpentInLogs: number
	startDatePlan?: string
	createdDate?: string
	closedDate?: string
	status?: string
}

interface ReportRow {
	id: string
	key: string
	employee: string
	taskTitle?: string
	tasksCount: number
	planned: number
	fact: number
	days: number
	plannedFormatted: string
	factFormatted: string
}

type TReportMode = 'byEmployee' | 'byTask'
const mode = ref<TReportMode>('byEmployee')

const employees = ref<any[]>([])
const employeesSelect = ref([])
const reportByEmployee = ref<ReportRow[]>([])
const employeeFilter = ref('all')
const reportByTask = ref<ReportRow[]>([])

const taskFilters = reactive({
	employees: [] as number[],
	status: 'all',
	period: { from: null as Date | null, to: null as Date | null },
})

// Сбрасываем отчёт и фильтры при смене режима
watch(mode, () => {
	isReportBuilt.value = false
	reportByEmployee.value = []
	reportByTask.value = []
	employeeFilter.value = 'all'
	taskFilters.status = 'all'
	taskFilters.period.from = null
	taskFilters.period.to = null
})

function toISO(date: Date) {
	return date.toISOString()
}

function startOfDay(date: Date) {
	const d = new Date(date)
	d.setHours(0, 0, 0, 0)
	return d
}

function endOfDay(date: Date) {
	const d = new Date(date)
	d.setHours(23, 59, 59, 999)
	return d
}

// Защита от кривого диапазона
watch(
	() => [taskFilters.period.from, taskFilters.period.to],
	([from, to]) => {
		if (from && to && from > to) taskFilters.period.to = null
	},
)

const totalsByEmployee = computed(() => {
	let tasks = 0,
		planned = 0,
		fact = 0
	reportByEmployee.value.forEach(r => {
		tasks += r.tasksCount
		planned += r.planned
		fact += r.fact
	})
	return {
		tasks,
		plannedFormatted: formatDuration(planned),
		factFormatted: formatDuration(fact),
	}
})

const totalsByTask = computed(() => {
	let tasks = 0,
		planned = 0,
		fact = 0
	reportByTask.value.forEach(r => {
		tasks += r.tasksCount
		planned += r.planned
		fact += r.fact
	})
	return {
		tasks,
		plannedFormatted: formatDuration(planned),
		factFormatted: formatDuration(fact),
	}
})

const api = <T,>(method: string, params: Record<string, any> = {}) =>
	new Promise<T>((resolve, reject) => {
		BX24.callMethod(method, params, (result: any) => {
			if (result.error()) reject(result.error())
			else resolve(result.data())
		})
	})

const OUTSOURCE_DEPARTMENT_ID = ['2']

async function loadUsers() {
	const filter: any = { UF_DEPARTMENT: OUTSOURCE_DEPARTMENT_ID }
	const department = await api<any>('department.get', {
		ID: OUTSOURCE_DEPARTMENT_ID,
	})
	const headId = department[0].UF_HEAD
	if (headId) filter['!ID'] = headId

	const users = await api<any>('user.get', { FILTER: filter })
	const normalized = users.map(u => ({
		title: `${u.NAME} ${u.LAST_NAME}`,
		value: u.ID,
	}))
	employees.value = users
	employeesSelect.value = [{ title: 'Все', value: 'all' }, ...normalized]
}

async function loadTasks() {
	const filter: Record<string, any> = {}

	if (mode.value === 'byEmployee' && employeeFilter.value !== 'all')
		filter.RESPONSIBLE_ID = employeeFilter.value

	if (mode.value === 'byTask' && taskFilters.employees.length)
		filter.RESPONSIBLE_ID = taskFilters.employees

	if (mode.value === 'byTask') {
		const statusMap: Record<string, number> = {
			new: 1,
			waiting_work: 2,
			in_progress: 3,
			completed: 5,
			deferred: 6,
		}
		if (taskFilters.status !== 'all')
			filter.STATUS = statusMap[taskFilters.status]
		if (taskFilters.period.from)
			filter['>=CLOSED_DATE'] = toISO(startOfDay(taskFilters.period.from))
		if (taskFilters.period.to)
			filter['<=CLOSED_DATE'] = toISO(endOfDay(taskFilters.period.to))
	}

	const data = await api<any>('tasks.task.list', {
		filter,
		select: [
			'ID',
			'TITLE',
			'RESPONSIBLE_ID',
			'TIME_ESTIMATE',
			'TIME_SPENT_IN_LOGS',
			'CLOSED_DATE',
			'CREATED_DATE',
			'START_DATE_PLAN',
			'STATUS',
			'DEADLINE',
		],
	})

	buildReport(data.tasks)
}

function buildReport(tasks: Task[]) {
	if (mode.value === 'byTask') {
		reportByTask.value = tasks.map(t => {
			const user = employees.value.find(u => u.ID === t.responsibleId)
			const name = user ? `${user.NAME} ${user.LAST_NAME}` : 'Имя не задано'

			return {
				key: `task-${t.id}`,
				employeeId: user.ID,
				employee: name,
				taskTitle: t.title,
				taskId: t.id,
				tasksCount: 1,
				planned: Number(t.timeEstimate) || 0,
				fact: Number(t.timeSpentInLogs) || 0,
				days: getTaskDurationDays(t),
				plannedFormatted: formatDuration(Number(t.timeEstimate) || 0),
				factFormatted: formatDuration(Number(t.timeSpentInLogs) || 0),
			}
		})
	} else {
		const map: Record<string, ReportRow> = {}
		employees.value.forEach(u => {
			if (employeeFilter.value !== 'all' && u.ID !== employeeFilter.value)
				return
			const name = `${u.NAME} ${u.LAST_NAME}`
			map[name] = {
				id: u.ID,
				key: `employee-${u.ID}`,
				employee: name,
				tasksCount: 0,
				planned: 0,
				fact: 0,
				days: 0,
				plannedFormatted: '0м',
				factFormatted: '0м',
			}
		})

		tasks.forEach(t => {
			const user = employees.value.find(u => u.ID === t.responsibleId)
			if (!user) return
			const name = `${user.NAME} ${user.LAST_NAME}`
			map[name].tasksCount++
			map[name].planned += Number(t.timeEstimate) || 0
			map[name].fact += Number(t.timeSpentInLogs) || 0
			map[name].days += getTaskDurationDays(t)
		})

		reportByEmployee.value = Object.values(map).map(r => {
			r.plannedFormatted = formatDuration(r.planned)
			r.factFormatted = formatDuration(r.fact)
			return r
		})
	}
}

const isLoading = ref(false)
const isAppLoading = ref(true)
const isReportBuilt = ref(false)

const build = async () => {
	isLoading.value = true
	try {
		await loadUsers()
		await loadTasks()
		isReportBuilt.value = true
	} finally {
		isLoading.value = false
	}
}

onMounted(async () => {
	await sleep(1000)
	await loadUsers()
	isAppLoading.value = false
})
</script>

<style scoped>
.overdue {
	font-weight: 600;
	color: #c00;
}
.app-start-box {
	width: 100%;
	min-height: 100dvh;
	display: grid;
	place-items: center;

}
.filters-box {
	margin-bottom: 1rem;
}
.mode-switch {
	display: flex;
	gap: 1rem;
}
.report-container {
	padding: 20px;
	font-family: Arial, sans-serif;
}
.report {
	overflow-x: auto;
}
.report-table {
	width: 100%;
	border-collapse: collapse;
	font-size: 14px;
}
.report-table th {
	text-align: left;
	padding: 10px;
	border-bottom: 2px solid #e5e5e5;
	font-weight: 600;
}
.report-table td {
	padding: 10px;
	border-bottom: 1px solid #eee;
}
.report-table tr:hover {
	background: #f7f9fb;
}
.total-row {
	background: #f1f3f5;
	font-weight: 600;
	border-top: 2px solid #ddd;
}
.table-row {
	transition:
		background-color 0.2s ease,
		transform 0.1s ease;
}
.table-row:hover {
	background-color: #e3f2fd;
}
.table-row:active {
	transform: scale(0.98);
	background-color: #cfe8fc;
}

.table-row td span {
	transition: all 0.2s ease;
	text-decoration: underline dotted;
	color: #1976d2;
}

.task-title {
	transition: all 0.2s ease;
	text-decoration: underline dotted;
	color: #1976d2;
}

.table-row td span:hover {
	color: #0d47a1;
	transform: scale(1.05);
}

.task-row {
	transition:
		background-color 0.2s ease,
		transform 0.1s ease;
}

.task-row:active {
	transform: scale(0.98);
	background-color: #cfe8fc;
}
</style>
