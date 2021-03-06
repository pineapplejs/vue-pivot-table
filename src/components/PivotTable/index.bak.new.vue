<template>
    <div style="margin: 20px;">
        <table>
            <template v-for="(tr, trIndex) in tableData">
                <tr
                    v-for="(_tr, _trIndex) in Array.isArray(tr[0])
                        ? tr.slice(1)
                        : [tr]"
                    :key="'' + trIndex + _trIndex"
                    :index="'' + trIndex + _trIndex"
                >
                    <template v-for="(td, tdIndex) in _tr">
                        <td
                            v-for="(_td, _tdIndex) in Array.isArray(td)
                                ? td.slice(1)
                                : [td]"
                            :key="
                                '' +
                                    trIndex +
                                    _trIndex +
                                    tdIndex +
                                    _tdIndex +
                                    ''
                            "
                            :index="
                                '' +
                                    trIndex +
                                    _trIndex +
                                    tdIndex +
                                    _tdIndex +
                                    ''
                            "
                            :rowspan="_td.rowspan"
                            :colspan="_td.colspan"
                        >
                            <div
                                draggable
                                :class="{
                                    summary: _td.isSummary,
                                    dragged: _td.dragged,
                                }"
                                :style="{
                                    'min-height': _getMinHeightByRowCount(
                                        _td.rowspan
                                    ),
                                }"
                                @dragstart="handleDragStart($event, _td)"
                                @drag="handleDrag"
                                @dragend="handleDragEnd($event, _td)"
                            >{{ _td.value }}</div>
                        </td>
                    </template>
                </tr>
            </template>
        </table>
    </div>
</template>

<script>
/**
 * pivot table
 */
import {
    mergeBaseInfo,
    deepClone,
    combine,
    combinePaths,
    convertPathToMap,
    findCategory,
    getHeightByCount,
    randomString
} from "./utils";
import { CELL_MIN_HEIGHT, SEPARATOR } from "./utils/constants";

// const IMG = new Image(85, 20);

/**
 * Feature:
 * - empty props checkout
 * - √ 新增列时，需要将新增列的计算后的结果和条件存入 calcData 中，并在 cell 中保存对应的 key
 * - 新增行时，直接通过key 和 条件来查找对应的计算列的汇总
 */

export default {
    name: "PivotTable",
    props: {
        rows: {
            type: Array,
            default: () => []
        },
        columns: {
            type: Array,
            default: () => []
        },
        values: {
            type: Array,
            default: () => []
        },
        // data
        data: {
            type: Array,
            default: () => []
        },
        // display column summary info. The default value is `false`
        columnSummary: {
            type: Array,
            default: () => []
        },
        // display row summary info. The default value is `false`
        rowSummary: {
            type: Array,
            default: () => []
        }
    },
    data: () => ({
        localRows: [],
        localColumns: [],
        localValues: [],
        tableData: [],
        localRowSummary: [],
        localColumnSummary: [],
        calcData: [],
        columnSummaryCalcData: [],
        // Separator
        Separator: SEPARATOR
    }),
    computed: {
        rowPaths() {
            const _paths = combinePaths(
                ...this.localRows.map(({ values }) => values),
                true
            );

            _paths.push("");

            return _paths;
        },
        colPaths() {
            const keys = this.localColumns
                .map(({ values }) => values)
                .concat([this.localValues.map(({ key }) => key)]);

            const _paths = combinePaths(...keys, true);

            _paths.push("");

            return _paths;
        },
        rowHeads() {
            return this.localRows.map(({ label }, index) => {
                return mergeBaseInfo({
                    value: label,
                    y: index,
                    _index: `0-${index}`,
                    rowspan:
                        this.localColumns.length +
                            Number(Boolean(this.localValues.length)) || 1
                });
            });
        },
        colHeads() {
            const _rows = this.localColumns.map(() => []);

            const valuesLen = this.localValues.length;

            if (valuesLen) {
                _rows.push([]);
            }

            this.colPaths.forEach((path, pathIndex) => {
                const values = path.split(this.Separator);
                const isSummary =
                    values.length !== this.localColumns.length + 1;
                const currPath = [];

                _rows.forEach((row, rowIndex) => {
                    const currVal = values[rowIndex] || "";
                    currPath.push(currVal);

                    const isValueKey =
                        valuesLen && currVal && rowIndex === _rows.length - 1;

                    const baseX = rowIndex;

                    const baseY =
                        rowIndex === 0 && this.localRows.length
                            ? this.localRows.length + pathIndex
                            : pathIndex;

                    const cell = mergeBaseInfo({
                        path: currPath.filter(item => !!item),
                        value: isValueKey
                            ? this.localValues.find(
                                  ({ key }) => key === currVal
                              ).label
                            : currVal,
                        x: baseX,
                        y: baseY,
                        _index: `${baseX}-${baseY}`,
                        isSummary
                    });

                    row.push(isSummary ? [cell] : cell);
                });
            });

            return _rows;
        },
        /*
            valueHeads() {
                const _valueHeads = [];

                const _paths = this.colPaths;

                const colCount = _paths.length || 1;

                for (let i = 0; i < colCount; i++) {
                    const _index = i % 2;
                    const currPath = _paths[i];
                    const path = currPath ? currPath.split(this.Separator) : [];
                    const currPathsLen =
                        _paths[i] !== undefined
                            ? _paths[i].split(this.Separator).length
                            : -1;
                    const baseX = this.localColumns.length;
                    const baseY = this.localRows.length + i;
                    const isSummary = currPathsLen !== this.localColumns.length + 1;

                    _valueHeads.push(
                        mergeBaseInfo({
                            path,
                            value: this.localValues[_index].label,
                            x: baseX,
                            y: baseY,
                            isSummary
                        })
                    );
                }

                return _valueHeads;
            },
        */
        rowHeadValues() {
            return this.rowPaths.map((path, index) => {
                const values = path.split(this.Separator);
                const currPath = [];
                const isSummary = this.localRows.length !== values.length;

                return this.localRows.map((...args) => {
                    const _index = args[1];
                    const _currVal = values[_index] || "";
                    const baseX =
                        this.localColumns.length +
                        +Boolean(this.localValues.length) +
                        index;
                    const baseY = _index;

                    currPath.push(_currVal);

                    return mergeBaseInfo({
                        path: currPath.filter(item => !!item),
                        value: _currVal,
                        x: baseX,
                        y: baseY,
                        _index: `${baseX}-${baseY}`,
                        isSummary
                    });
                });
            });
        },
        dataValues() {
            // conditions of col head
            const colConditions = convertPathToMap(
                this.colPaths,
                this.localColumns
                    .map(({ key }) => key)
                    .concat(this.localValues.length ? ["value"] : [])
            );

            // conditions of row-head
            const rowConditions = convertPathToMap(
                this.rowPaths,
                this.localRows.map(({ key }) => key)
            );

            // Note: if there are no props.rows or props.column, push an empty object
            !colConditions.length && colConditions.push({});
            !rowConditions.length && rowConditions.push({});

            // draw data
            const _dataValues = rowConditions.map(
                (rowCondition, rowConditionIndex) => {
                    const _row = colConditions.map(
                        (colCondition, colConditionIndex) => {
                            // the condition of current cell
                            const conditions = Object.assign(
                                {},
                                rowCondition,
                                colCondition
                            );

                            // console.log(Object.keys(conditions));

                            // const isSummary =
                            //     Object.keys(conditions).length !==
                            //     this.pathKeys.length + 1;
                            // console.log(isSummary);
                            const isSummary = !conditions.value;
                            // console.log(conditions.value, "-", isSummary);

                            const _key = conditions.value;

                            delete conditions.value;

                            let cellData = [];

                            // filter the data
                            const filterData = this._filterData(
                                conditions,
                                this.localData
                            );

                            const baseX =
                                this.localColumns.length +
                                +Boolean(this.localValues.length) +
                                rowConditionIndex;

                            const baseY =
                                this.localRows.length + colConditionIndex;

                            // the filtered data is passed to the `handle` of `props.values`, return calculated data.
                            // Note: there is no `handle` in the `this.localValues`.
                            if (
                                this.localColumns.length &&
                                this.localRows.length &&
                                !this.localValues.length
                            ) {
                                // render empty cell
                                cellData.push(
                                    mergeBaseInfo({
                                        path: conditions,
                                        x: baseX,
                                        y: baseY,
                                        value: "",
                                        isSummary,
                                        _index: `${baseX}-${baseY}`
                                    })
                                );
                            } else {
                                if (isSummary) {
                                    cellData = [
                                        mergeBaseInfo({
                                            path: conditions,
                                            x: baseX,
                                            y: baseY,
                                            key: _key,
                                            value: "",
                                            isSummary,
                                            _index: `${baseX}-${baseY}`
                                        })
                                    ];
                                } else {
                                    const _value = this.values.find(
                                        ({ key }) => key === _key
                                    );

                                    cellData = mergeBaseInfo({
                                        path: conditions,
                                        x: baseX,
                                        y: baseY,
                                        key: _key,
                                        value: _value.handle
                                            ? _value.handle(filterData)
                                            : "",
                                        isSummary,
                                        _index: `${baseX}-${baseY}`
                                    });
                                }
                            }

                            return cellData;
                        }
                    );

                    return _row;
                }
            );

            console.log("_dataValues", _dataValues);

            return _dataValues.filter(item => item.length);
        },
        pathKeys() {
            return this.localRows
                .map(({ key }) => key)
                .concat(this.localColumns.map(({ key }) => key));
        },
        // monitor all props value changes
        watchAllProps() {
            const {
                rows,
                columns,
                values,
                data,
                columnSummary,
                rowSummary
            } = this;

            return {
                rows,
                columns,
                values,
                data,
                columnSummary,
                rowSummary
            };
        }
    },
    created() {
        this.init();
    },
    methods: {
        init() {
            if (this.rows.length || this.columns.length || this.values) {
                this.handleDataClone();
                this.setValuesToColAndRow();
                this.combineTable();
                this.handleCalcData();
                this.handleSummary();
            } else {
                console.warn(
                    "[Warn]: props.rows, props.columns, props.values at least one is not empty."
                );
                // this.tableData = [];
            }
        },
        // clone data
        handleDataClone() {
            this.localColumns = deepClone(this.columns);
            this.localRows = deepClone(this.rows);
            // Note: this.localValues can't clone handle function.
            this.localValues = deepClone(this.values);
            this.localData = Object.freeze(this.data);

            this.localRowSummary = this.rowSummary.map(item =>
                Object.assign(
                    {},
                    { ...item },
                    { key: item.key || randomString() }
                )
            );

            this.localColumnSummary = this.columnSummary.map(item =>
                Object.assign(
                    {},
                    { ...item },
                    { key: item.key || randomString() }
                )
            );
        },
        // set the `values` attribute to rows and columns
        setValuesToColAndRow() {
            const columnKeys = this.localColumns.map(({ key }) => key);
            const rowKeys = this.localRows.map(({ key }) => key);
            const columnValues = findCategory(columnKeys, this.localData);
            const rowValues = findCategory(rowKeys, this.localData);

            this.localColumns.forEach(column => {
                const { key, values } = column;
                this.$set(column, "values", values || columnValues[key] || []);
            });

            this.localRows.forEach(row => {
                const { key, values } = row;
                this.$set(row, "values", values || rowValues[key] || []);
            });
        },
        // combine table heads
        handleCombineHeads() {
            let combineColHeads = [...this.colHeads];
            combineColHeads[0] = combineColHeads[0] || [];
            combineColHeads[0].unshift(...this.rowHeads);
            combineColHeads = combineColHeads.filter(item => item.length);

            // console.log("6. combineColHeads", combineColHeads);

            this.combineHeads = combineColHeads;
        },
        // combine table values
        handleCombineValues() {
            // values
            const combineValues = [];

            const valueRowCount =
                this.dataValues.length || this.rowHeadValues.length;

            for (let i = 0; i < valueRowCount; i++) {
                const _currRowHeadValue = this.rowHeadValues[i] || [];
                const _currValue = this.dataValues[i] || [];
                const _row = [..._currRowHeadValue, ..._currValue];

                if (_row[0].isSummary) {
                    _row.forEach(item => {
                        if (!Array.isArray(item)) {
                            item.isSummary = true;
                        }
                    });
                    combineValues.push([_row]);
                } else {
                    combineValues.push(_row);
                }
            }

            // console.log("7. valueRowCount", valueRowCount);

            // console.log("8. combineValues", combineValues);

            this.combineValues = combineValues;
        },
        // combination table
        combineTable() {
            this.handleCombineHeads();
            this.handleCombineValues();
            this.tableData = [...this.combineHeads, ...this.combineValues];

            // console.log("colPaths", this.colPaths);
            // combinePaths(
            //     ...this.localColumns.map(({ values }) => values),
            //     isSummary
            // );
            // const keys = this.localColumns.map(({ values }) => values);
            // keys.push(this.localValues.map(({ key }) => key));
            // console.log(keys);
            // console.log(combinePaths(...keys, true));
            // console.log("1. rowHeads", this.rowHeads);
            // console.log("2. colHeads", this.colHeads);
            // console.log("3. valueHeads", this.valueHeads);
            // console.log("4. rowHeadValues", this.rowHeadValues);
            // console.log("5. dataValues", this.dataValues);

            // console.log("this.computedColHeads", this.computedColHeads);

            // console.log("this.computedValueHeads", this.computedValueHeads);

            // heads
            // let combineColHeads = [...this.colHeads, this.valueHeads];

            // console.log("9. tabledata", this.tableData);
        },
        handleSummary() {
            if (this.localColumnSummary.length) {
                this.localColumnSummary.forEach(item => {
                    this.handleAddColumnSummary();
                });
            }

            if (this.localRowSummary.length) {
                this.localRowSummary.forEach(item => {
                    this.handleAddRowSummary();
                });
            }
        },
        // set calc data
        handleCalcData() {
            const _colPaths = combinePaths(
                ...this.localColumns.map(({ values }) => values)
            );

            const _colKeys = this.localColumns.map(({ key }) => key);

            const _rowPaths = combinePaths(
                ...this.localRows.map(({ values }) => values)
            );

            const _rowKeys = this.localRows.map(({ key }) => key);

            // conditions of col head
            const colConditions = convertPathToMap(_colPaths, _colKeys);

            // conditions of row-head
            const rowConditions = convertPathToMap(_rowPaths, _rowKeys);

            // Note: if there are no props.rows or props.column, push an empty object
            !colConditions.length && colConditions.push({});
            !rowConditions.length && rowConditions.push({});

            // draw data
            this.calcData = Object.freeze(
                rowConditions
                    .map((rowCondition, rowConditionIndex) =>
                        colConditions.map((colCondition, colConditionIndex) => {
                            // the condition of current cell
                            const conditions = Object.assign(
                                {},
                                rowCondition,
                                colCondition
                            );

                            const cellData = {};

                            // filter the data
                            const filterData = this._filterData(
                                conditions,
                                this.localData
                            );

                            // empty cell
                            const isEmptyCell =
                                this.localColumns.length &&
                                this.localRows.length &&
                                !this.localValues.length;

                            const _values = {};

                            this.values.forEach(({ key, handle }) => {
                                _values[key] = isEmptyCell
                                    ? ""
                                    : handle
                                    ? handle(filterData)
                                    : "";
                            });

                            return Object.assign({}, conditions, _values);
                        })
                    )
                    .filter(item => item.length)
                    .flat()
            );
        },
        // add column summary
        handleAddColumnSummary() {
            this.$nextTick(() => {
                const { key, label, handle } = this.localColumnSummary.shift();

                this.combineHeads.forEach((headRow, headRowIndex) => {
                    headRow.forEach(cell => {
                        if (Array.isArray(cell)) {
                            const isLastRow =
                                headRowIndex === this.combineHeads.length - 1;

                            cell.push(
                                Object.assign({}, cell[0], {
                                    y: "to-be-calc",
                                    value: isLastRow ? label : cell[0].value
                                })
                            );
                        }
                    });
                });

                // console.log("this.combineHeads", this.combineHeads);
                console.log("this.combineValues", this.combineValues);
                // console.log(this.calcData, "this.calcData");

                // const _handle = (data) => {
                //     let clicks = 0;
                //     let downloads = 0;

                //     data.forEach(({ click, download }) => {
                //         clicks += +click;
                //         downloads += +download;
                //     });

                //     const result = clicks / downloads;

                //     return Number.isNaN(result)
                //         ? ""
                //         : (result * 100).toFixed(2) + "%";
                // };

                const _calcData = [];

                const handleValueRow = (row = []) => {
                    row.forEach(cell => {
                        if (Array.isArray(cell)) {
                            const _filterData = this._filterData(
                                cell[0].path,
                                this.calcData
                            );
                            // console.log(_filterData);
                            const value = handle(_filterData);

                            cell.push(
                                Object.assign({}, cell[0], {
                                    y: "to-be-calc",
                                    value,
                                    key
                                })
                            );

                            _calcData.push(
                                Object.assign({}, cell[0].path, {
                                    [key]: value
                                })
                            );
                        }
                    });
                };

                this.combineValues.forEach((valueRow, valueRowIndex) => {
                    if (Array.isArray(valueRow[0])) {
                        valueRow.forEach(row => {
                            handleValueRow(row);
                        });
                    } else {
                        handleValueRow(valueRow);
                    }
                });

                // this.calcData.push(..._calcData);
                this.columnSummaryCalcData = Object.freeze(
                    this.columnSummaryCalcData.concat(_calcData)
                );

                // console.log(
                //     "columnSummaryCalcData",
                //     this.columnSummaryCalcData
                // );
                this.tableData = [...this.combineHeads, ...this.combineValues];
            });
        },
        // add row summary
        handleAddRowSummary() {
            // const handle = (data) => {};
            const { key, label, handle } = this.localRowSummary.shift();

            this.$nextTick(() => {
                this.combineValues.forEach(valueRow => {
                    if (Array.isArray(valueRow[0])) {
                        const _cloneRow = JSON.parse(
                            JSON.stringify(valueRow[0])
                        );

                        // console.log(_cloneRow, "_cloneRow");

                        _cloneRow.forEach((cell, index) => {
                            if (Array.isArray(cell)) {
                                cell.slice(1).forEach(_cell => {
                                    // console.log(_cell.key, _cell.path);
                                    const _filterData = this._filterData(
                                        _cell.path,
                                        this.columnSummaryCalcData
                                    );
                                    // console.log(_filterData);
                                    _cell.value = "[" + index + "]";
                                });
                            } else {
                                const _filterData = this._filterData(
                                    cell.path,
                                    this.calcData
                                ).map(item => item[cell.key]);
                                cell.value = handle(_filterData);
                                // console.log(cell.path, cell.key);
                            }
                        });
                        valueRow.push(_cloneRow);
                    }
                });

                // console.log("this.combineValues", this.combineValues);

                this.tableData = [...this.combineHeads, ...this.combineValues];
            });
        },
        saveCalcData(condition, key, value) {},
        // recalc the position(x-y)
        // drag cell start
        handleDragStart(e, data) {
            const { target, dataTransfer } = e;

            data.dragged = true;

            dataTransfer.effectAllowed = "copy";

            dataTransfer.setData("data", JSON.stringify(data));

            dataTransfer.setDragImage(target, 0, 0);

            // IMG.src = dragTip;

            // const canvas = document.createElement("canvas");

            // const ctx = canvas.getContext("2d");

            // canvas.width = 170;
            // canvas.height = 42;

            // ctx.fillStyle = "white";
            // ctx.fillRect(0, 0, canvas.width, canvas.height);

            // ctx.fillStyle = "red";
            // ctx.font = "14px serif";

            // ctx.fillText("推拽生成图表", 10, 24);

            // IMG.src = canvas.toDataURL("image/png");

            // IMG.onload = () => {
            //     dataTransfer.setDragImage(IMG, 0, 0);
            // };
        },
        // dragging cell
        handleDrag(e) {
            e.dataTransfer.effectAllowed = "copy";
        },
        // dragging cell end
        handleDragEnd(e, data) {
            data.dragged = false;
            this.$emit(
                "on-dragend",
                { ...data },
                {
                    rows: [...this.localRows],
                    columns: [...this.localColumns],
                    data: [...this.calcData]
                }
            );
        },
        _filterData(conditions, data) {
            return data.filter(data => {
                let status = true;

                for (let key in conditions) {
                    if (conditions[key] !== data[key]) {
                        status = false;
                        return;
                    }
                }

                return status;
            });
        },
        // get min height by rowspan
        _getMinHeightByRowCount(count) {
            return getHeightByCount(count);
        }
    },
    watch: {
        watchAllProps() {
            this.init();
            this.$emit("on-change");
        }
    }
};
</script>

<style lang="scss" scoped>
table {
    border-collapse: collapse;
    border-spacing: 0;
    border: none;

    td {
        border: 1px solid #ccc;
        padding: 0;
        vertical-align: middle;
        box-sizing: border-box;

        > div {
            display: flex;
            align-items: center;
            justify-content: center;
            box-sizing: border-box;
            padding: 5px;
            text-align: center;
            white-space: nowrap;
            width: 100%;
            height: 100%;
            min-height: 36px;
            cursor: default;

            &.dragging {
                user-select: none;
            }

            &.summary {
                background: #eee;
            }

            &.active {
                // outline: 1px solid #aaa;
                // background: #ccc;
                outline: 1px dashed rgba(0, 0, 0, 0.5);
                background: #bbb;
            }

            &.dragged {
                outline: 1px dashed rgba(0, 0, 0, 0.5);
                background: #bbb;
            }
        }
    }
}
</style>
