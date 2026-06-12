<script setup lang="ts">
import type Calculator from "@/calculator"
import type { ActionDetail } from "~/game"
import ItemIcon from "@@/components/ItemIcon/index.vue"
import { Delete, Search, Setting } from "@element-plus/icons-vue"
import { ElMessage, ElMessageBox } from "element-plus"
import { DecomposeCalculator, TransmuteCalculator } from "@/calculator/alchemy"
import { ManufactureCalculator } from "@/calculator/manufacture"
import { getGameDataApi, getItemDetailOf, getPriceOf } from "@/common/apis/game"
import { getManualPriceOf, setPriceApi } from "@/common/apis/price"
import { useMemory } from "@/common/composables/useMemory"
import * as Format from "@/common/utils/format"
import { COIN_HRID } from "@/pinia/stores/game"
import { usePriceStore } from "@/pinia/stores/price"

const props = defineProps<{
  data: Calculator[]
  loading: boolean
}>()

const emit = defineEmits<{
  (e: "show-detail", row: Calculator): void
  (e: "delete-strategy", row: Calculator): void
}>()

const priceStore = usePriceStore()
const { t } = useI18n()
const STRATEGY_MAIN_ICON_SIZE = 60
const STRATEGY_SUB_ICON_SIZE = STRATEGY_MAIN_ICON_SIZE * 0.92
const STRATEGY_SUB_SOURCE_ICON_SIZE = STRATEGY_MAIN_ICON_SIZE * 0.84
const expandedStrategyKey = ref<string>()
const priceEditorMode = ref<"input" | "output">("input")
const currentPriceRow = ref<Calculator>()
const currentProductPriceConfigList = ref<Calculator["productPriceConfigList"]>([])
const currentIngredientPriceConfigList = ref<Calculator["ingredientPriceConfigList"]>([])
interface StrategyPriceRow {
  hrid: string
  level?: number
  price: number
  manual: boolean
  immutable?: boolean
}
type AcquisitionType = "decompose" | "transmute"
type ProductUsageType = AcquisitionType | "manufacture"
interface AcquisitionOption {
  key: string
  type: AcquisitionType
  calculator: Calculator
  label: string
}
interface ProductUsageOption {
  key: string
  type: ProductUsageType
  calculator: Calculator
  label: string
}
interface AcquisitionSourcePriceRow {
  key: string
  type: "ingredient" | "product"
  priceRow: StrategyPriceRow
}
const acquisitionTargetKey = ref<string>()
const acquisitionTargetRow = ref<StrategyPriceRow>()
const acquisitionFromType = ref<AcquisitionType>()
const acquisitionSourceKey = ref<string>()
const acquisitionSourceCalculator = ref<Calculator>()
const acquisitionSourcePriceConfigList = ref<StrategyPriceRow[]>([])
const acquisitionSourceProductPriceConfigList = ref<StrategyPriceRow[]>([])
const productUsageTargetKey = ref<string>()
const productUsageTargetRow = ref<AcquisitionSourcePriceRow>()
const productUsageOptionKey = ref<string>()
const productUsageOption = ref<ProductUsageOption>()
const productUsageCalculator = ref<Calculator>()
const productUsageIngredientPriceConfigList = ref<StrategyPriceRow[]>([])
const productUsageProductPriceConfigList = ref<StrategyPriceRow[]>([])
const acquisitionOptionsCache = new Map<string, AcquisitionOption[]>()
const productUsageOptionsCache = new Map<string, ProductUsageOption[]>()
const acquisitionExpandRowKeys = computed(() => acquisitionTargetKey.value ? [acquisitionTargetKey.value] : [])
const productUsageExpandRowKeys = computed(() => productUsageTargetKey.value ? [productUsageTargetKey.value] : [])
const acquisitionSourceDisplayPriceRows = computed<AcquisitionSourcePriceRow[]>(() => {
  const ingredients = acquisitionSourcePriceConfigList.value.map((priceRow, index) => ({
    key: `ingredient-${index}-${getPriceRowKey(priceRow)}`,
    type: "ingredient" as const,
    priceRow
  }))
  const products: AcquisitionSourcePriceRow[] = []
  const calculator = acquisitionSourceCalculator.value
  const target = acquisitionTargetRow.value
  if (calculator && target) {
    acquisitionSourceProductPriceConfigList.value.slice(0, getDirectAcquisitionProductCount(calculator, acquisitionFromType.value)).forEach((priceRow, index) => {
      const product = calculator.productListWithPrice[index]
      if (!product || priceRowMatchesTarget(product, target) || getProductUnits(product) <= 1e-8) {
        return
      }
      products.push({
        key: `product-${index}-${getPriceRowKey(priceRow)}`,
        type: "product",
        priceRow
      })
    })
  }
  return ingredients.length ? [ingredients[0], ...products, ...ingredients.slice(1)] : products
})
const productUsageDisplayPriceRows = computed<AcquisitionSourcePriceRow[]>(() => {
  const target = productUsageTargetRow.value?.priceRow
  const ingredients = productUsageIngredientPriceConfigList.value
    .filter(priceRow => !target || !priceRowMatchesTarget(priceRow, target))
    .map((priceRow, index) => ({
      key: `usage-ingredient-${index}-${getPriceRowKey(priceRow)}`,
      type: "ingredient" as const,
      priceRow
    }))
  const calculator = productUsageCalculator.value
  const productCount = calculator ? getDirectProductUsageProductCount(calculator, productUsageOption.value?.type) : productUsageProductPriceConfigList.value.length
  const products = productUsageProductPriceConfigList.value.slice(0, productCount).map((priceRow, index) => ({
    key: `usage-product-${index}-${getPriceRowKey(priceRow)}`,
    type: "product" as const,
    priceRow
  }))
  return [...ingredients, ...products]
})
type StrategyInfoKey = "project" | "volume" | "unitsHour" | "unitsDay" | "incomeHour" | "incomeDay" | "profitHour" | "profitDay" | "expHour" | "expDay" | "rate"
const defaultStrategyInfo: StrategyInfoKey[] = ["project", "volume", "unitsHour", "incomeHour", "profitHour", "expHour", "rate"]
const strategyInfoKeys: StrategyInfoKey[] = ["project", "volume", "unitsHour", "unitsDay", "incomeHour", "incomeDay", "profitHour", "profitDay", "expHour", "expDay", "rate"]
const visibleStrategyInfo = useMemory("dashboard-strategy-card-visible-info", defaultStrategyInfo) as Ref<string[]>
const strategyInfoOptions = computed<{ key: StrategyInfoKey, label: string }[]>(() => [
  { key: "project", label: t("动作") },
  { key: "volume", label: t("成交量(1h)") },
  { key: "unitsHour", label: t("单位 / h") },
  { key: "unitsDay", label: t("单位 / 天") },
  { key: "incomeHour", label: t("收入 / h") },
  { key: "incomeDay", label: t("收入 / 天") },
  { key: "profitHour", label: t("利润 / h") },
  { key: "profitDay", label: t("利润 / 天") },
  { key: "expHour", label: t("经验 / h") },
  { key: "expDay", label: t("经验 / 天") },
  { key: "rate", label: t("利润率") }
])
const hasStrategyMetaInfo = computed(() => visibleStrategyInfo.value.some(key => key === "project" || key === "volume"))
const hasStrategyStatsInfo = computed(() => visibleStrategyInfo.value.some(key => ["unitsHour", "unitsDay", "incomeHour", "incomeDay", "profitHour", "profitDay", "expHour", "expDay", "rate"].includes(key)))

visibleStrategyInfo.value = normalizeStrategyInfo(visibleStrategyInfo.value)

function normalizeStrategyInfo(keys: string[]) {
  const normalized = keys
    .map((key) => {
      if (key === "profit") {
        return "profitHour"
      }
      if (key === "exp") {
        return "expHour"
      }
      return key
    })
    .filter((key): key is StrategyInfoKey => strategyInfoKeys.includes(key as StrategyInfoKey))
  const unique = Array.from(new Set(normalized))
  if ((keys.includes("profit") || keys.includes("exp")) && !unique.includes("incomeHour")) {
    unique.splice(Math.min(2, unique.length), 0, "incomeHour")
  }
  return unique.length ? unique : [...defaultStrategyInfo]
}

function showStrategyInfo(key: StrategyInfoKey) {
  return visibleStrategyInfo.value.includes(key)
}

function setStrategyInfo(key: StrategyInfoKey, visible: boolean) {
  const next = new Set(normalizeStrategyInfo(visibleStrategyInfo.value))
  visible ? next.add(key) : next.delete(key)
  visibleStrategyInfo.value = Array.from(next)
}

function formatPerDay(value: unknown) {
  return typeof value === "number" && Number.isFinite(value) ? Format.money(value * 24) : "-"
}

function getOutputUnitsPerHour(row: Calculator) {
  const rowLevel = (row as any).calculator?.enhanceLevel ?? (row as any).enhanceLevel ?? 0
  const matchingProduct = row.productListWithPrice.find(item => item.hrid === row.hrid && (item.level ?? 0) === rowLevel)
    ?? row.productListWithPrice.find(item => item.hrid === row.hrid)
    ?? row.productListWithPrice[0]
  const units = matchingProduct?.countPH ?? row.result.gainPH
  return typeof units === "number" && Number.isFinite(units) && units >= 0 ? units : undefined
}

function formatUnits(value: unknown) {
  return typeof value === "number" && Number.isFinite(value) ? Format.number(value, 3) : "-"
}

function formatUnitsPerHour(row: Calculator) {
  return formatUnits(getOutputUnitsPerHour(row))
}

function formatUnitsPerDay(row: Calculator) {
  const units = getOutputUnitsPerHour(row)
  return typeof units === "number" ? formatUnits(units * 24) : "-"
}

function formatVolume1h(row: any) {
  const hrid = row?.hrid
  const level = row?.calculator?.enhanceLevel ?? row?.enhanceLevel ?? 0
  const vol = getPriceOf(hrid, level).vol ?? -1
  return vol < 0 ? "-" : Format.number(vol)
}

function getPriceRowKey(row: Pick<StrategyPriceRow, "hrid" | "level">) {
  return `${row.hrid}-${row.level ?? 0}`
}

function getProductUnits(product: { countPH?: number, count?: number, rate?: number }) {
  return product.countPH ?? ((product.count ?? 0) * (product.rate ?? 1))
}

function priceRowMatchesTarget(row: Pick<StrategyPriceRow, "hrid" | "level">, target: StrategyPriceRow) {
  return row.hrid === target.hrid && (row.level ?? 0) === (target.level ?? 0)
}

function productMatchesTarget(product: { hrid: string, level?: number, countPH?: number, count?: number, rate?: number }, target: StrategyPriceRow) {
  return priceRowMatchesTarget(product, target) && getProductUnits(product) > 1e-8
}

function getAcquisitionSourceDisplayRowKey(row: AcquisitionSourcePriceRow) {
  return row.key
}

function getDirectAcquisitionProductCount(calculator: Calculator, type?: AcquisitionType) {
  const alchemyDetail = (calculator as any).item?.alchemyDetail
  if (type === "transmute") {
    return alchemyDetail?.transmuteDropTable?.length ?? calculator.productListWithPrice.length
  }
  if (type === "decompose") {
    return ((calculator as any).enhanceLevel > 0 ? 1 : 0) + (alchemyDetail?.decomposeItems?.length ?? 0)
  }
  return calculator.productListWithPrice.length
}

function getDirectProductUsageProductCount(calculator: Calculator, type?: ProductUsageType) {
  if (type === "manufacture") {
    return (calculator as any).actionItem?.outputItems?.length ?? calculator.productListWithPrice.length
  }
  return getDirectAcquisitionProductCount(calculator, type)
}

function getAcquisitionTypeLabel(type: AcquisitionType) {
  return type === "decompose" ? t("分解") : t("转化")
}

function getManufactureProjectLabel(action: string) {
  switch (action) {
    case "cheesesmithing":
      return t("锻造")
    case "crafting":
      return t("制造")
    case "tailoring":
      return t("裁缝")
    case "cooking":
      return t("烹饪")
    case "brewing":
      return t("冲泡")
    default:
      return action
  }
}

function getActionFromActionDetail(actionDetail: ActionDetail) {
  return actionDetail.type.substring(actionDetail.type.lastIndexOf("/") + 1)
}

function makeAcquisitionOption(type: AcquisitionType, calculator: Calculator): AcquisitionOption {
  const sourceLevel = (calculator as any).enhanceLevel || 0
  const sourceName = t(getItemDetailOf(calculator.hrid).name)
  return {
    key: `${type}:${calculator.hrid}:${sourceLevel}`,
    type,
    calculator,
    label: `${sourceName}${sourceLevel ? ` +${sourceLevel}` : ""}`
  }
}

function getAcquisitionOptions(target: StrategyPriceRow) {
  const key = getPriceRowKey(target)
  const cached = acquisitionOptionsCache.get(key)
  if (cached) {
    return cached
  }

  if (target.hrid === COIN_HRID) {
    acquisitionOptionsCache.set(key, [])
    return []
  }

  const options: AcquisitionOption[] = []
  for (const item of Object.values(getGameDataApi().itemDetailMap)) {
    const calculators: Array<{ type: AcquisitionType, calculator: Calculator }> = [
      { type: "decompose", calculator: new DecomposeCalculator({ hrid: item.hrid }) },
      { type: "transmute", calculator: new TransmuteCalculator({ hrid: item.hrid }) }
    ]

    for (const { type, calculator } of calculators) {
      try {
        if (!calculator.available) {
          continue
        }
        if (calculator.productListWithPrice.some(product => productMatchesTarget(product, target))) {
          options.push(makeAcquisitionOption(type, calculator))
        }
      } catch {
        // Some game-data combinations are intentionally unavailable for a calculator.
      }
    }
  }

  options.sort((a, b) => getAcquisitionTypeLabel(a.type).localeCompare(getAcquisitionTypeLabel(b.type)) || a.label.localeCompare(b.label))
  acquisitionOptionsCache.set(key, options)
  return options
}

function getProductUsageOptions(target: StrategyPriceRow) {
  const key = getPriceRowKey(target)
  const cached = productUsageOptionsCache.get(key)
  if (cached) {
    return cached
  }

  if (target.hrid === COIN_HRID) {
    productUsageOptionsCache.set(key, [])
    return []
  }

  const manufactureActions = ["cheesesmithing", "crafting", "tailoring", "cooking", "brewing"]
  const options: ProductUsageOption[] = []
  for (const actionDetail of Object.values(getGameDataApi().actionDetailMap) as ActionDetail[]) {
    const action = getActionFromActionDetail(actionDetail)
    const usesTarget = actionDetail.inputItems?.some(input => input.itemHrid === target.hrid)
      || actionDetail.upgradeItemHrid === target.hrid
    if (!manufactureActions.includes(action) || !usesTarget || !actionDetail.outputItems?.length) {
      continue
    }

    try {
      const outputHrid = actionDetail.outputItems[0].itemHrid
      const calculator = new ManufactureCalculator({
        hrid: outputHrid,
        project: getManufactureProjectLabel(action),
        action: action as any
      })
      if (!calculator.available || !calculator.ingredientList.some(ingredient => priceRowMatchesTarget(ingredient, target))) {
        continue
      }
      options.push({
        key: `manufacture:${actionDetail.hrid}`,
        type: "manufacture",
        calculator,
        label: `${getManufactureProjectLabel(action)}: ${t(getItemDetailOf(outputHrid).name)}`
      })
    } catch {
      // Skip recipes that cannot be represented by the manufacture calculator.
    }
  }

  try {
    const calculator = new TransmuteCalculator({ hrid: target.hrid })
    if (calculator.available) {
      options.push({
        key: `transmute:${target.hrid}`,
        type: "transmute",
        calculator,
        label: getAcquisitionTypeLabel("transmute")
      })
    }
  } catch {
    // Some items cannot be transmuted.
  }

  try {
    const calculator = new DecomposeCalculator({ hrid: target.hrid, enhanceLevel: target.level || 0 })
    if (calculator.available) {
      options.push({
        key: `decompose:${target.hrid}:${target.level || 0}`,
        type: "decompose",
        calculator,
        label: getAcquisitionTypeLabel("decompose")
      })
    }
  } catch {
    // Some items cannot be decomposed.
  }

  productUsageOptionsCache.set(key, options)
  return options
}

function resetProductUsagePanel() {
  productUsageTargetKey.value = undefined
  productUsageTargetRow.value = undefined
  productUsageOptionKey.value = undefined
  productUsageOption.value = undefined
  productUsageCalculator.value = undefined
  productUsageIngredientPriceConfigList.value = []
  productUsageProductPriceConfigList.value = []
}

function resetAcquisitionPanel() {
  acquisitionTargetKey.value = undefined
  acquisitionTargetRow.value = undefined
  acquisitionFromType.value = undefined
  acquisitionSourceKey.value = undefined
  acquisitionSourceCalculator.value = undefined
  acquisitionSourcePriceConfigList.value = []
  acquisitionSourceProductPriceConfigList.value = []
  resetProductUsagePanel()
}

function getActiveAcquisitionTypeOptions() {
  if (!acquisitionTargetRow.value) {
    return []
  }
  const activeTypes = new Set(getAcquisitionOptions(acquisitionTargetRow.value).map(option => option.type))
  return (["transmute", "decompose"] as AcquisitionType[]).filter(type => activeTypes.has(type))
}

function getActiveAcquisitionSourceOptions() {
  if (!acquisitionTargetRow.value || !acquisitionFromType.value) {
    return []
  }
  return getAcquisitionOptions(acquisitionTargetRow.value).filter(option => option.type === acquisitionFromType.value)
}

function setAcquisitionSource(sourceKey?: string | number | boolean) {
  resetProductUsagePanel()
  const key = typeof sourceKey === "string" ? sourceKey : undefined
  const option = getActiveAcquisitionSourceOptions().find(option => option.key === key) ?? getActiveAcquisitionSourceOptions()[0]
  acquisitionSourceKey.value = option?.key
  acquisitionSourceCalculator.value = option?.calculator
  acquisitionSourcePriceConfigList.value = option ? getPriceConfigList(option.calculator, "ingredient") : []
  acquisitionSourceProductPriceConfigList.value = option ? getPriceConfigList(option.calculator, "product") : []
}

function getActiveProductUsageOptions() {
  const target = productUsageTargetRow.value?.priceRow
  return target ? getProductUsageOptions(target) : []
}

function setProductUsageOption(optionKey?: string | number | boolean) {
  const key = typeof optionKey === "string" ? optionKey : undefined
  const option = getActiveProductUsageOptions().find(option => option.key === key) ?? getActiveProductUsageOptions()[0]
  productUsageOptionKey.value = option?.key
  productUsageOption.value = option
  productUsageCalculator.value = option?.calculator
  productUsageIngredientPriceConfigList.value = option ? getPriceConfigList(option.calculator, "ingredient") : []
  productUsageProductPriceConfigList.value = option ? getPriceConfigList(option.calculator, "product") : []
}

function hasProductUsageOptions(row: AcquisitionSourcePriceRow) {
  return row.type === "product" && getProductUsageOptions(row.priceRow).length > 0
}

function toggleProductUsagePanel(row: AcquisitionSourcePriceRow) {
  if (row.type !== "product") {
    return
  }

  if (productUsageTargetKey.value === row.key) {
    resetProductUsagePanel()
    return
  }

  const options = getProductUsageOptions(row.priceRow)
  if (!options.length) {
    ElMessage.info(t("暂无来源"))
    return
  }

  productUsageTargetKey.value = row.key
  productUsageTargetRow.value = row
  setProductUsageOption(options[0].key)
}

function isProductUsageExpanded(row: AcquisitionSourcePriceRow) {
  return productUsageTargetKey.value === row.key
}

function setAcquisitionType(type?: string | number | boolean) {
  acquisitionFromType.value = type === "decompose" || type === "transmute" ? type : undefined
  setAcquisitionSource()
}

function toggleAcquisitionPanel(priceRow: StrategyPriceRow) {
  const key = getPriceRowKey(priceRow)
  if (acquisitionTargetKey.value === key) {
    resetAcquisitionPanel()
    return
  }

  const options = getAcquisitionOptions(priceRow)
  if (!options.length) {
    ElMessage.info(t("暂无来源"))
    return
  }

  acquisitionTargetKey.value = key
  acquisitionTargetRow.value = priceRow
  setAcquisitionType(options.some(option => option.type === "transmute") ? "transmute" : options[0].type)
}

function isAcquisitionExpanded(priceRow: StrategyPriceRow) {
  return acquisitionTargetKey.value === getPriceRowKey(priceRow)
}

function getCatalyst(row: Calculator) {
  return (row as any).catalyst as string | undefined
}

function getStrategyKey(row: Calculator) {
  return `${row.id}-${row.catalystRank ?? "none"}-${row.project}`
}

function isExpanded(row: Calculator, mode?: "input" | "output") {
  return expandedStrategyKey.value === getStrategyKey(row) && (!mode || priceEditorMode.value === mode)
}

function resetPriceEditor() {
  expandedStrategyKey.value = undefined
  priceEditorMode.value = "input"
  currentPriceRow.value = undefined
  currentIngredientPriceConfigList.value = []
  currentProductPriceConfigList.value = []
  resetAcquisitionPanel()
}

function getPriceConfigList(row: Calculator, type: "product" | "ingredient") {
  return row[`${type}ListWithPrice`].map((item, i) => {
    const priceConfig = row[`${type}PriceConfigList`][i]
    const priceType = type === "ingredient" ? "ask" : "bid"
    const hasManualPrice = getManualPriceOf(item.hrid, item.level)?.[priceType]?.manual
    const manualPrice = getManualPriceOf(item.hrid, item.level)?.[priceType]?.manualPrice
    const price = priceConfig?.immutable ? priceConfig.price! : hasManualPrice ? manualPrice! : item.marketPrice
    return {
      hrid: item.hrid,
      level: item.level,
      price,
      manual: priceConfig?.manual || hasManualPrice || false,
      immutable: priceConfig?.immutable
    }
  })
}

function openPriceEditor(row: Calculator, mode: "input" | "output") {
  resetAcquisitionPanel()
  expandedStrategyKey.value = getStrategyKey(row)
  priceEditorMode.value = mode
  currentPriceRow.value = row
  currentIngredientPriceConfigList.value = getPriceConfigList(row, "ingredient")
  currentProductPriceConfigList.value = getPriceConfigList(row, "product")
}

async function togglePriceEditor(row: Calculator, mode: "input" | "output") {
  if (isExpanded(row, mode)) {
    resetPriceEditor()
    return
  }

  if (!priceStore.activated) {
    try {
      await ElMessageBox.confirm(t("是否确定开启自定义价格？"), t("需先开启自定义价格"), {
        confirmButtonText: t("确定"),
        cancelButtonText: t("取消"),
        closeOnClickModal: true
      })
      priceStore.setActivated(true)
    } catch {
      return
    }
  }

  openPriceEditor(row, mode)
}

function onConfirmPrice() {
  if (!currentPriceRow.value) {
    return
  }

  try {
    if (productUsageCalculator.value) {
      setPriceApi(productUsageCalculator.value, productUsageIngredientPriceConfigList.value, productUsageProductPriceConfigList.value)
    }
    if (acquisitionSourceCalculator.value) {
      setPriceApi(acquisitionSourceCalculator.value, acquisitionSourcePriceConfigList.value, acquisitionSourceProductPriceConfigList.value)
    }
    setPriceApi(currentPriceRow.value, currentIngredientPriceConfigList.value, currentProductPriceConfigList.value)
    resetPriceEditor()
  } catch (e: any) {
    ElMessage.error(e.message)
  }
}

watch(() => props.data, () => {
  if (expandedStrategyKey.value && !props.data.some(row => getStrategyKey(row) === expandedStrategyKey.value)) {
    resetPriceEditor()
  }
})
</script>

<template>
  <el-card class="strategies-card" v-loading="loading">
    <template #header>
      <div class="strategies-header">
        <div class="title">
          {{ t('策略集') }}
        </div>
        <el-popover placement="bottom-end" trigger="click" width="230">
          <template #reference>
            <el-button class="strategy-settings-button" :icon="Setting" circle size="small" aria-label="Strategy Settings" />
          </template>
          <div class="strategy-settings">
            <div
              v-for="option in strategyInfoOptions"
              :key="option.key"
              class="strategy-settings-row"
            >
              <span>{{ option.label }}</span>
              <el-checkbox
                :model-value="showStrategyInfo(option.key)"
                :aria-label="option.label"
                @change="setStrategyInfo(option.key, Boolean($event))"
              />
            </div>
          </div>
        </el-popover>
      </div>
    </template>

    <el-empty v-if="!loading && !data.length" :description="t('暂无策略')" />

    <div v-else>
      <div class="strategy-list">
        <div
          v-for="row in data"
          :key="getStrategyKey(row)"
          class="strategy-row"
          :class="{ 'is-expanded': isExpanded(row) }"
        >
          <div class="strategy-icons">
            <ItemIcon class="strategy-main-icon" :hrid="row.hrid" :width="STRATEGY_MAIN_ICON_SIZE" :height="STRATEGY_MAIN_ICON_SIZE" />
            <ItemIcon v-if="getCatalyst(row)" class="catalyst-icon" :hrid="`/items/${getCatalyst(row)}`" :width="22" :height="22" />
          </div>

          <div class="strategy-body">
            <div class="strategy-title-row">
              <div class="strategy-title">
                {{ row.result.name }}
              </div>
            </div>
            <div v-if="hasStrategyMetaInfo" class="strategy-meta">
              <span v-if="showStrategyInfo('project')">{{ row.project }}</span>
              <span v-if="showStrategyInfo('volume')">{{ t('成交量(1h)') }}: {{ formatVolume1h(row) }}</span>
            </div>
            <div v-if="hasStrategyStatsInfo" class="strategy-stats">
              <span v-if="showStrategyInfo('unitsHour')">{{ t('单位 / h') }}: {{ formatUnitsPerHour(row) }}</span>
              <span v-if="showStrategyInfo('unitsDay')">{{ t('单位 / 天') }}: {{ formatUnitsPerDay(row) }}</span>
              <span v-if="showStrategyInfo('incomeHour')" :class="row.hasManualPrice ? 'manual' : ''">{{ t('收入 / h') }}: {{ row.result.incomePHFormat }}</span>
              <span v-if="showStrategyInfo('incomeDay')" :class="row.hasManualPrice ? 'manual' : ''">{{ t('收入 / 天') }}: {{ formatPerDay(row.result.incomePH) }}</span>
              <span v-if="showStrategyInfo('profitHour')" :class="row.hasManualPrice ? 'manual' : ''">{{ t('利润 / h') }}: {{ row.result.profitPHFormat }}</span>
              <span v-if="showStrategyInfo('profitDay')" :class="row.hasManualPrice ? 'manual' : ''">{{ t('利润 / 天') }}: {{ row.result.profitPDFormat || formatPerDay(row.result.profitPH) }}</span>
              <span v-if="showStrategyInfo('expHour')">{{ t('经验 / h') }}: {{ row.result.expPHFormat }}</span>
              <span v-if="showStrategyInfo('expDay')">{{ t('经验 / 天') }}: {{ formatPerDay(row.result.expPH) }}</span>
              <span v-if="showStrategyInfo('rate')">{{ t('利润率') }}: {{ row.result.profitRateFormat }}</span>
            </div>
          </div>

          <div class="strategy-actions">
            <el-button
              class="strategy-action strategy-action--input"
              :class="{ 'is-expanded': isExpanded(row, 'input') }"
              size="small"
              aria-label="From"
              @click="togglePriceEditor(row, 'input')"
            >
              From
            </el-button>
            <el-button
              class="strategy-action strategy-action--output"
              :class="{ 'is-expanded': isExpanded(row, 'output') }"
              size="small"
              aria-label="To"
              @click="togglePriceEditor(row, 'output')"
            >
              To
            </el-button>
            <el-button class="strategy-action strategy-action--delete" :icon="Delete" circle size="small" type="danger" :aria-label="t('删除')" @click="emit('delete-strategy', row)" />
            <el-button class="strategy-action strategy-action--view" :icon="Search" circle size="small" :aria-label="t('查看')" @click="emit('show-detail', row)" />
          </div>

          <el-collapse-transition>
            <div v-if="isExpanded(row)" class="strategy-price-panel">
              <div v-if="priceEditorMode === 'input'" class="strategy-price-section strategy-price-section--input">
                <el-table
                  :data="currentIngredientPriceConfigList"
                  :row-key="getPriceRowKey"
                  :expand-row-keys="acquisitionExpandRowKeys"
                  class="strategy-input-table"
                  size="small"
                >
                  <el-table-column type="expand" width="1" class-name="strategy-source-expand-column">
                    <template #default="{ row: priceRow }">
                      <div v-if="isAcquisitionExpanded(priceRow)" class="strategy-source-panel">
                        <div class="strategy-source-controls">
                          <div class="strategy-source-title">
                            <span>From</span>
                          </div>
                          <el-select
                            :model-value="acquisitionFromType"
                            placeholder="From"
                            size="small"
                            class="strategy-source-select"
                            @change="setAcquisitionType"
                          >
                            <el-option
                              v-for="type in getActiveAcquisitionTypeOptions()"
                              :key="type"
                              :label="getAcquisitionTypeLabel(type)"
                              :value="type"
                            />
                          </el-select>
                          <el-select
                            :model-value="acquisitionSourceKey"
                            :placeholder="t('物品')"
                            size="small"
                            class="strategy-source-select strategy-source-item-select"
                            @change="setAcquisitionSource"
                          >
                            <el-option
                              v-for="option in getActiveAcquisitionSourceOptions()"
                              :key="option.key"
                              :label="option.label"
                              :value="option.key"
                            />
                          </el-select>
                        </div>

                        <el-table
                          :data="acquisitionSourceDisplayPriceRows"
                          :row-key="getAcquisitionSourceDisplayRowKey"
                          :expand-row-keys="productUsageExpandRowKeys"
                          size="small"
                          class="strategy-source-table"
                        >
                          <el-table-column type="expand" width="1" class-name="strategy-product-usage-expand-column">
                            <template #default="{ row: sourcePriceRow }">
                              <div v-if="isProductUsageExpanded(sourcePriceRow)" class="strategy-product-usage-panel">
                                <div class="strategy-source-controls">
                                  <div class="strategy-source-title">
                                    <span>Use</span>
                                  </div>
                                  <el-select
                                    :model-value="productUsageOptionKey"
                                    placeholder="Use"
                                    size="small"
                                    class="strategy-source-select strategy-source-item-select"
                                    @change="setProductUsageOption"
                                  >
                                    <el-option
                                      v-for="option in getActiveProductUsageOptions()"
                                      :key="option.key"
                                      :label="option.label"
                                      :value="option.key"
                                    />
                                  </el-select>
                                </div>

                                <el-table
                                  :data="productUsageDisplayPriceRows"
                                  :row-key="getAcquisitionSourceDisplayRowKey"
                                  size="small"
                                  class="strategy-product-usage-table"
                                >
                                  <el-table-column label="" width="90" align="center" class-name="strategy-sub-source-icon-column">
                                    <template #default="{ row: usagePriceRow }">
                                      <ItemIcon class="strategy-sub-source-icon" :hrid="usagePriceRow.priceRow.hrid" :width="STRATEGY_SUB_SOURCE_ICON_SIZE" :height="STRATEGY_SUB_SOURCE_ICON_SIZE" />
                                    </template>
                                  </el-table-column>
                                  <el-table-column prop="price" :label="t('市场价格')" min-width="120">
                                    <template #default="{ row: usagePriceRow }">
                                      <div v-if="usagePriceRow.priceRow.hrid === COIN_HRID">
                                        {{ Format.price(usagePriceRow.priceRow.price) }}
                                      </div>
                                      <div v-else>
                                        {{ Format.price(getPriceOf(usagePriceRow.priceRow.hrid, usagePriceRow.priceRow.level).ask) }} / {{ Format.price(getPriceOf(usagePriceRow.priceRow.hrid, usagePriceRow.priceRow.level).bid) }}
                                      </div>
                                    </template>
                                  </el-table-column>
                                  <el-table-column :label="t('自定义价格')" min-width="160">
                                    <template #default="{ row: usagePriceRow }">
                                      <div class="strategy-manual-input">
                                        <el-checkbox v-show="usagePriceRow.priceRow.hrid !== COIN_HRID" v-model="usagePriceRow.priceRow.manual" />
                                        <el-input-number v-show="usagePriceRow.priceRow.manual" v-model="usagePriceRow.priceRow.price" :controls="false" size="small" />
                                      </div>
                                    </template>
                                  </el-table-column>
                                </el-table>
                              </div>
                            </template>
                          </el-table-column>
                          <el-table-column label="" width="90" align="center" class-name="strategy-sub-source-icon-column">
                            <template #default="{ row: sourcePriceRow }">
                              <ItemIcon class="strategy-sub-source-icon" :hrid="sourcePriceRow.priceRow.hrid" :width="STRATEGY_SUB_SOURCE_ICON_SIZE" :height="STRATEGY_SUB_SOURCE_ICON_SIZE" />
                            </template>
                          </el-table-column>
                          <el-table-column width="54" class-name="strategy-product-usage-button-column">
                            <template #default="{ row: sourcePriceRow }">
                              <el-button
                                v-if="hasProductUsageOptions(sourcePriceRow)"
                                class="strategy-product-usage-button"
                                :class="{ 'is-expanded': isProductUsageExpanded(sourcePriceRow) }"
                                size="small"
                                aria-label="To"
                                @click="toggleProductUsagePanel(sourcePriceRow)"
                              >
                                To
                              </el-button>
                            </template>
                          </el-table-column>
                          <el-table-column prop="price" :label="t('市场价格')" min-width="120" class-name="strategy-market-price-column">
                            <template #default="{ row: sourcePriceRow }">
                              <div v-if="sourcePriceRow.priceRow.hrid === COIN_HRID">
                                {{ Format.price(sourcePriceRow.priceRow.price) }}
                              </div>
                              <div v-else>
                                {{ Format.price(getPriceOf(sourcePriceRow.priceRow.hrid, sourcePriceRow.priceRow.level).ask) }} / {{ Format.price(getPriceOf(sourcePriceRow.priceRow.hrid, sourcePriceRow.priceRow.level).bid) }}
                              </div>
                            </template>
                          </el-table-column>
                          <el-table-column :label="t('自定义价格')" min-width="160">
                            <template #default="{ row: sourcePriceRow }">
                              <div class="strategy-manual-input">
                                <el-checkbox v-show="sourcePriceRow.priceRow.hrid !== COIN_HRID" v-model="sourcePriceRow.priceRow.manual" />
                                <el-input-number v-show="sourcePriceRow.priceRow.manual" v-model="sourcePriceRow.priceRow.price" :controls="false" size="small" />
                              </div>
                            </template>
                          </el-table-column>
                        </el-table>
                      </div>
                    </template>
                  </el-table-column>
                  <el-table-column label="" width="90" align="center" class-name="strategy-sub-icon-column">
                    <template #default="{ row: priceRow }">
                      <ItemIcon class="strategy-sub-icon" :hrid="priceRow.hrid" :width="STRATEGY_SUB_ICON_SIZE" :height="STRATEGY_SUB_ICON_SIZE" />
                    </template>
                  </el-table-column>
                  <el-table-column width="54" class-name="strategy-source-button-column">
                    <template #default="{ row: priceRow }">
                      <el-button
                        class="strategy-source-button"
                        :class="{ 'is-expanded': isAcquisitionExpanded(priceRow) }"
                        size="small"
                        aria-label="From"
                        @click="toggleAcquisitionPanel(priceRow)"
                      >
                        From
                      </el-button>
                    </template>
                  </el-table-column>
                  <el-table-column prop="price" :label="t('市场价格')" min-width="120" class-name="strategy-market-price-column">
                    <template #default="{ row: priceRow }">
                      <div v-if="priceRow.hrid === COIN_HRID">
                        {{ Format.price(priceRow.price) }}
                      </div>
                      <div v-else>
                        {{ Format.price(getPriceOf(priceRow.hrid, priceRow.level).ask) }} / {{ Format.price(getPriceOf(priceRow.hrid, priceRow.level).bid) }}
                      </div>
                    </template>
                  </el-table-column>

                  <el-table-column :label="t('自定义价格')" min-width="160">
                    <template #default="{ row: priceRow }">
                      <div class="strategy-manual-input">
                        <el-checkbox v-show="priceRow.hrid !== COIN_HRID" v-model="priceRow.manual" />
                        <el-input-number v-show="priceRow.manual" v-model="priceRow.price" :controls="false" size="small" />
                      </div>
                    </template>
                  </el-table-column>
                </el-table>
              </div>

              <div v-else class="strategy-price-section strategy-price-section--output">
                <el-table :data="currentProductPriceConfigList" size="small">
                  <el-table-column label="" width="90" align="center">
                    <template #default="{ row: priceRow }">
                      <ItemIcon class="strategy-sub-icon" :hrid="priceRow.hrid" :width="STRATEGY_SUB_ICON_SIZE" :height="STRATEGY_SUB_ICON_SIZE" />
                    </template>
                  </el-table-column>
                  <el-table-column prop="price" :label="t('市场价格')" min-width="120">
                    <template #default="{ row: priceRow }">
                      <div v-if="priceRow.hrid === COIN_HRID">
                        {{ Format.price(priceRow.price) }}
                      </div>
                      <div v-else>
                        {{ Format.price(getPriceOf(priceRow.hrid, priceRow.level).ask) }} / {{ Format.price(getPriceOf(priceRow.hrid, priceRow.level).bid) }}
                      </div>
                    </template>
                  </el-table-column>
                  <el-table-column :label="t('自定义价格')" min-width="160">
                    <template #default="{ row: priceRow }">
                      <div class="strategy-manual-input">
                        <el-checkbox v-show="priceRow.hrid !== COIN_HRID" v-model="priceRow.manual" />
                        <el-input-number v-show="priceRow.manual" v-model="priceRow.price" :controls="false" size="small" />
                      </div>
                    </template>
                  </el-table-column>
                </el-table>
              </div>

              <div class="strategy-price-footer">
                <el-button type="primary" size="small" @click="onConfirmPrice">
                  {{ t('保存') }}
                </el-button>
              </div>
            </div>
          </el-collapse-transition>
        </div>
      </div>
    </div>
  </el-card>
</template>

<style lang="scss" scoped>
.strategies-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;

  .title {
    font-weight: 600;
  }
}

.strategy-settings-button {
  margin-left: 0;
}

.strategy-settings {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.strategy-settings-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  color: var(--el-text-color-regular);
  font-size: 14px;
  line-height: 1.2;
}

.strategy-settings-row :deep(.el-checkbox) {
  height: 18px;
  margin-right: 0;
}

.strategy-settings-row :deep(.el-checkbox__label) {
  display: none;
}

.strategy-list {
  display: flex;
  flex-direction: column;
}

.strategies-card {
  --strategy-card-padding: 8px;
  --strategy-tree-blue: #1682ff;
  --strategy-tree-blue-light: #79bdff;
}

.strategies-card :deep(.el-card__header) {
  padding: var(--strategy-card-padding);
}

.strategies-card :deep(.el-card__body) {
  padding: var(--strategy-card-padding);
}

.strategies-card :deep(.el-scrollbar__bar) {
  display: none;
}

.strategy-row {
  position: relative;
  isolation: isolate;
  display: grid;
  grid-template-columns: 150px minmax(0, 1fr);
  gap: 12px;
  min-height: 88px;
  padding: 0 42px 10px 0;
  border-bottom: 1px solid var(--el-border-color-lighter);

  &:first-child {
    padding-top: 0;
  }

  &:last-child {
    border-bottom: none;
    padding-bottom: 0;
  }

  &.is-expanded {
    padding-bottom: 12px;
  }
}

.strategy-icons {
  position: relative;
  z-index: 4;
  display: flex;
  align-items: flex-start;
  align-self: start;
  min-width: 150px;
  padding-top: 0;
  padding-left: 0;
}

.catalyst-icon {
  position: absolute;
  right: 0;
  bottom: 6px;
  opacity: 0.75;
}

.strategy-body {
  min-width: 0;
  padding-top: 8px;
}

.strategy-title-row {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
}

.strategy-title {
  min-width: 0;
  overflow: hidden;
  flex: 0 1 auto;
  font-weight: 600;
  line-height: 1.3;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.strategy-meta,
.strategy-stats {
  display: flex;
  flex-wrap: wrap;
  gap: 6px 12px;
  margin-top: 5px;
  color: var(--el-text-color-secondary);
  font-size: 12px;
  line-height: 1.35;
}

.strategy-actions {
  display: contents;
}

.strategy-action {
  position: absolute;
  z-index: 5;
  margin-left: 0;
  min-width: 48px;
  height: 24px;
  padding: 0 9px;
  border-color: transparent;
  border-radius: 4px;
  color: #111;
  font-size: 12px;
  font-weight: 700;
  line-height: 1;

  &:hover,
  &:focus {
    color: #111;
    border-color: transparent;
  }
}

.strategy-action--input {
  top: 0;
  left: 68px;
  background: #5f86ff;

  &.is-expanded {
    box-shadow: 0 0 0 2px var(--el-color-primary-light-7);
  }

  &:hover,
  &:focus {
    background: #7397ff;
  }
}

.strategy-action--output {
  top: 32px;
  left: 68px;
  background: #d9a529;

  &.is-expanded {
    box-shadow: 0 0 0 2px var(--el-color-warning-light-5);
  }

  &:hover,
  &:focus {
    background: #e1b13d;
  }
}

.strategy-action--delete {
  top: 0;
  right: 0;
}

.strategy-action--view {
  right: 0;
  top: 42px;
}

.manual {
  color: #409eff;
}

.strategy-price-panel {
  grid-column: 1 / -1;
  margin-top: 8px;
  padding: 0;
}

.strategy-price-section--input {
  position: relative;
}

.strategy-price-section--input::before {
  content: "";
  position: absolute;
  top: -61px;
  bottom: 42px;
  left: 2px;
  z-index: 3;
  width: 1px;
  border-radius: 0 0 8px 8px;
  background: var(--strategy-tree-blue);
  pointer-events: none;
}

.strategy-manual-input {
  display: flex;
  align-items: center;
  gap: 8px;
  min-height: 26px;
}

.strategy-manual-input :deep(.el-input-number) {
  width: 116px;
}

.strategy-source-button {
  margin-left: 0;
  min-width: 48px;
  height: 24px;
  padding: 0 9px;
  border-color: transparent;
  border-radius: 4px;
  background: #5f86ff;
  color: #111;
  font-size: 12px;
  font-weight: 700;
  line-height: 1;

  &:hover,
  &:focus {
    border-color: transparent;
    background: #7397ff;
    color: #111;
  }
}

.strategy-input-table,
.strategy-source-table,
.strategy-product-usage-table,
.strategy-price-section--output :deep(.el-table) {
  --el-table-row-hover-bg-color: transparent;
}

.strategy-input-table :deep(.el-table__body tr:hover > td.el-table__cell),
.strategy-input-table :deep(.el-table__body tr.hover-row > td.el-table__cell),
.strategy-source-table :deep(.el-table__body tr:hover > td.el-table__cell),
.strategy-source-table :deep(.el-table__body tr.hover-row > td.el-table__cell),
.strategy-product-usage-table :deep(.el-table__body tr:hover > td.el-table__cell),
.strategy-product-usage-table :deep(.el-table__body tr.hover-row > td.el-table__cell),
.strategy-price-section--output :deep(.el-table__body tr:hover > td.el-table__cell),
.strategy-price-section--output :deep(.el-table__body tr.hover-row > td.el-table__cell) {
  background-color: transparent !important;
}

.strategy-input-table :deep(.el-table__body td.el-table__cell),
.strategy-source-table :deep(.el-table__body td.el-table__cell),
.strategy-product-usage-table :deep(.el-table__body td.el-table__cell),
.strategy-price-section--output :deep(.el-table__body td.el-table__cell) {
  border-bottom: none;
}

.strategy-input-table :deep(.strategy-source-button-column) {
  padding: 0;
  vertical-align: top;
}

.strategy-input-table :deep(.strategy-source-button-column .cell) {
  display: flex;
  align-items: flex-start;
  justify-content: flex-start;
  height: 100%;
  padding: 4px 0 0 0;
  overflow: visible;
  text-overflow: clip;
}

.strategy-input-table :deep(.el-table__body-wrapper .strategy-market-price-column),
.strategy-source-table :deep(.el-table__body-wrapper .strategy-market-price-column) {
  vertical-align: top;
}

.strategy-input-table :deep(.el-table__body-wrapper .strategy-market-price-column .cell),
.strategy-source-table :deep(.el-table__body-wrapper .strategy-market-price-column .cell) {
  min-height: 28px;
  padding: 4px 0 0 2px;
  line-height: 24px;
  white-space: nowrap;
}

.strategy-input-table :deep(.el-table__body-wrapper .strategy-sub-icon-column .cell) {
  position: relative;
  overflow: visible;
}

.strategy-input-table :deep(.el-table__body-wrapper .strategy-sub-icon-column .cell::before) {
  content: "";
  position: absolute;
  top: 0;
  bottom: 50%;
  left: 1px;
  z-index: 0;
  width: 42px;
  border-bottom: 1px solid var(--strategy-tree-blue);
  border-left: 1px solid var(--strategy-tree-blue);
  border-bottom-left-radius: 8px;
  pointer-events: none;
}

.strategy-sub-icon {
  position: relative;
  z-index: 1;
  display: block;
  margin-left: 14px;
  margin-right: auto;
}

.strategy-sub-source-icon {
  position: relative;
  z-index: 1;
  display: block;
  margin: 0 auto;
}

.strategy-input-table :deep(.strategy-source-expand-column) {
  width: 1px !important;
  min-width: 1px !important;
  max-width: 1px !important;
  padding: 0 !important;
  overflow: hidden;
}

.strategy-input-table :deep(.strategy-source-expand-column .cell) {
  display: none;
  width: 0 !important;
  min-width: 0 !important;
  padding: 0;
  overflow: hidden;
}

.strategy-input-table :deep(.el-table__expand-icon) {
  display: none;
}

.strategy-input-table :deep(.el-table__expanded-cell) {
  background: transparent;
  padding: 0 0 10px;
}

.strategy-source-button.is-expanded {
  box-shadow: 0 0 0 2px var(--el-color-primary-light-7);
}

.strategy-source-table :deep(.strategy-product-usage-button-column) {
  padding: 0;
  vertical-align: top;
}

.strategy-source-table :deep(.strategy-product-usage-button-column .cell) {
  display: flex;
  align-items: flex-start;
  justify-content: flex-start;
  height: 100%;
  padding: 4px 0 0;
  overflow: visible;
  text-overflow: clip;
}

.strategy-source-table :deep(.el-table__body-wrapper .strategy-sub-source-icon-column .cell),
.strategy-product-usage-table :deep(.el-table__body-wrapper .strategy-sub-source-icon-column .cell) {
  position: relative;
  overflow: visible;
}

.strategy-source-table :deep(.el-table__body-wrapper .strategy-sub-source-icon-column .cell::before),
.strategy-product-usage-table :deep(.el-table__body-wrapper .strategy-sub-source-icon-column .cell::before) {
  content: "";
  position: absolute;
  top: 0;
  bottom: 50%;
  left: -19px;
  z-index: 0;
  width: 52px;
  border-bottom: 1px solid var(--strategy-tree-blue-light);
  border-left: 1px solid var(--strategy-tree-blue-light);
  border-bottom-left-radius: 8px;
  pointer-events: none;
}

.strategy-product-usage-button {
  margin-left: 0;
  min-width: 48px;
  height: 24px;
  padding: 0 9px;
  border-color: transparent;
  border-radius: 4px;
  background: #d9a529;
  color: #111;
  font-size: 12px;
  font-weight: 700;
  line-height: 1;

  &:hover,
  &:focus {
    border-color: transparent;
    background: #e1b13d;
    color: #111;
  }
}

.strategy-source-table :deep(.strategy-product-usage-expand-column) {
  width: 1px !important;
  min-width: 1px !important;
  max-width: 1px !important;
  padding: 0 !important;
  overflow: hidden;
}

.strategy-source-table :deep(.strategy-product-usage-expand-column .cell) {
  display: none;
  width: 0 !important;
  min-width: 0 !important;
  padding: 0;
  overflow: hidden;
}

.strategy-source-table :deep(.el-table__expand-icon) {
  display: none;
}

.strategy-source-table :deep(.el-table__expanded-cell) {
  background: transparent;
  padding: 0 0 10px;
}

.strategy-product-usage-button.is-expanded {
  box-shadow: 0 0 0 2px var(--el-color-primary-light-7);
}

.strategy-source-panel {
  position: relative;
  border-top: 1px solid var(--el-border-color-lighter);
  margin: 0;
  padding: 8px 0 0 24px;
}

.strategy-product-usage-panel {
  position: relative;
  border-top: 1px solid var(--el-border-color-lighter);
  margin: 0;
  padding: 8px 0 0 24px;
}

.strategy-source-panel::before,
.strategy-product-usage-panel::before {
  content: "";
  position: absolute;
  top: 8px;
  bottom: 40px;
  left: 6px;
  z-index: 1;
  width: 1px;
  border-radius: 0 0 8px 8px;
  background: var(--strategy-tree-blue-light);
  pointer-events: none;
}

.strategy-source-controls {
  display: flex;
  align-items: center;
  flex-wrap: nowrap;
  gap: 8px;
  margin-bottom: 8px;
}

.strategy-source-title {
  display: flex;
  align-items: center;
  gap: 6px;
  flex: 0 0 auto;
  min-width: 0;
  color: var(--el-text-color-regular);
  font-size: 13px;
  font-weight: 600;
}

.strategy-source-select {
  width: 150px;
  flex: 0 0 150px;
}

.strategy-source-item-select {
  width: 260px;
  flex: 0 1 260px;
}

.strategy-price-footer {
  display: flex;
  justify-content: center;
  margin-top: 10px;
}

@media (max-width: 640px) {
  .strategy-row {
    padding-right: 40px;
  }

  .strategy-price-panel {
    padding: 0;
  }

  .strategy-source-controls {
    flex-wrap: wrap;
  }

  .strategy-source-select,
  .strategy-source-item-select {
    flex: 1 1 150px;
    width: auto;
  }
}
</style>
