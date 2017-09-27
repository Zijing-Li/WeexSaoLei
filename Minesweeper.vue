<template>
	<div class="content">
		<div class="content-header">
		</div>
		<div class="row"
			 :style="{height:gridSize}"
			 v-for="(row, rowIndex) in mineList">
			<div class="grid"
				 :style="{width:gridSize, height:gridSize}"
				 v-for="(grid, columnIndex) in row"
				 @click="clickGrid(rowIndex, columnIndex)"
				 @longpress="longpressGrid(rowIndex, columnIndex)">
				<div class="grid-bg" :style="{backgroundColor:gridStatus[grid.status].backgroundColor}"></div>
				<image class="grid-image"
					   resize="contain"
					   v-if="grid.status==1||grid.status==3||(grid.status==2&&grid.mine)"
					   :src="getGridImage(grid)">
				</image>
				<text class="grid-text" v-if="grid.mine==false&&grid.status==2&&grid.mineCount>0">{{grid.mineCount}}</text>
			</div>
		</div>
		<div class="content-footer">
			<text class="reset-button" v-if="gameStatus!=0" @click="resetGame">{{gameStatus==1?'再玩一次':'重来'}}</text>
		</div>
	</div>
</template>

<script>
	const modal = weex.requireModule('modal')
	var timer = null
    export default {
		computed: {
			gridSize: function () {
				return 750.0/this.columnNumber + 'px'
			}
		},
        data: {
		    started: false,
            rowNumber: 12,
			columnNumber: 9,
			mineCount: 20,
			gameStatus: 0, // 0进行中 1赢 2输
			exposeCount: 0, // 格子打开数
            // 雷
			mineList:[],
			// 格子状态
			gridStatus: [
				{
				    // 隐藏
				    status: 'hide',
					backgroundColor: '#28384B'
				},
				{
				    // 标识
					status: 'tag',
					backgroundColor: '#28384B'
				},
				{
				    // 打开
				    status: 'expose',
					backgroundColor: 'orange'
				},
				{
				    // 爆炸
				    status: 'explode',
					backgroundColor: 'orange'
				}
			],
			clickRow: -1,
			clickColumn: -1,
			clickCount: 0,
			tagImage: require('@/assets/img/qizi.png'),				// 旗子
			mineImage: require('@/assets/img/mine_1.png'),			// 雷：默认
			explodeMineImage: require('@/assets/img/mine_2.png'),	// 雷：爆炸
		},
        methods: {
		    checkSuccess: function () {
				// 检查是否赢了，打开的格子等于未布雷的格子数
				if (this.exposeCount==this.rowNumber*this.columnNumber-this.mineCount) {
				    this.success()
				}
			},
			showAll: function () {
				for (var rowIndex in this.mineList) {
				    var row = this.mineList[rowIndex]
					for (var columnIndex in row) {
					    var grid = row[columnIndex]
						if (grid.status!=3) {
							grid.status = 2
						}
					}
				}
			},
			success: function () {
				this.gameStatus = 1
				this.showAll()
				toast('你赢了！', 3)
			},
			failed: function () {
				this.gameStatus = 2
				this.showAll()
				toast('你输了！', 3)
			},
		    getGridImage: function (grid) {
				switch (grid.status) {
					case 1:
					    return this.tagImage
					case 2:
					    if (grid.mine) {
					        return this.mineImage
						}
						return ''
					case 3:
					    return this.explodeMineImage
					default:
						return ''
				}
			},
			clickGrid: function(rowIndex, columnIndex) {
		        if (this.clickRow!=rowIndex||this.clickColumn!=columnIndex) {
					clearTimeout(timer)
					if (this.clickCount > 0) {
						this.clickCount = 0
						this.clickRow = -1
						this.clickColumn = -1
					    return
					}
					this.clickCount = 0
				}
				this.clickCount++
				if (this.clickCount == 1) {
					this.clickRow = rowIndex
					this.clickColumn = columnIndex
					var self = this
					timer = setTimeout(function () {
						self.singleClickGrid(rowIndex, columnIndex)
						self.clickCount = 0
						self.clickRow = -1
						self.clickColumn = -1
					}, 300)
				} else {
					clearTimeout(timer)
					this.clickCount = 0
					this.clickRow = -1
					this.clickColumn = -1
					this.longpressGrid(rowIndex, columnIndex)
				}
			},
			singleClickGrid: function(rowIndex, columnIndex) {
				if (this.gameStatus!=0) {
					return
				}
				if (!this.started) {
					this.startGame(rowIndex, columnIndex, false)
				}
				var row = this.mineList[rowIndex]
				var grid = row[columnIndex]
				switch (grid.status) {
					case 0:
						grid.status=2
						if (grid.mine) {
							grid.status=3
							this.failed()
						} else if (grid.mineCount==0) {
							this.tryTouchGrid(rowIndex, columnIndex, [rowIndex*this.columnNumber+columnIndex])
						}
						this.exposeCount++
						this.checkSuccess()
						break
					case 1:
						grid.status=0
						break
					default:
						return
				}
				this.mineList.splice(rowIndex, 1, row)
			},
			longpressGrid: function(rowIndex, columnIndex) {
				if (this.gameStatus!=0) {
					return
				}
				var row = this.mineList[rowIndex]
				var grid = row[columnIndex]
				switch (grid.status) {
					case 0:
						grid.status=1
						break
					default:
						return
				}
				if (!this.started) {
					this.startGame(rowIndex, columnIndex, true)
				}
				this.mineList.splice(rowIndex, 1, row)
			},
			tryTouchGrid: function (rowIndex, columnIndex, touchedGrid) {
		        // 找到这个点周围所有没有雷的点
				for (var i=rowIndex-1; i<=rowIndex+1; i++) {
					if (i<0 || i>=this.mineList.length) {
						continue
					}
					for (var j=columnIndex-1; j<=columnIndex+1; j++) {
					    var point
						if (j<0 || j>=this.mineList[i].length || contains(touchedGrid, point=i*this.columnNumber+j)) {
							continue
						}
						touchedGrid.push(point)
						var grid = this.mineList[i][j]
						if (grid.mine==false&&grid.mineCount==0) {
					        // 处理斜角空白格
					        if (rowIndex!=i&&columnIndex!=j) {
								var g1 = this.mineList[i][columnIndex]
								var g2 = this.mineList[rowIndex][j]
								if (g1.mineCount>0&&g2.mineCount>0) {
									continue
								}
							}
							grid.status = 2
							this.exposeCount++
							// 对没有布雷且周围都没有布雷的点递归查找
							this.tryTouchGrid(i, j, touchedGrid)
						}
					}
				}
			},
			startGame: function (rowIndex, columnIndex, isLongpress) {
				this.started = true
				// 随机布雷
				this.randomMine(rowIndex, columnIndex, isLongpress)
				// 标记周围雷数量
				for (var i=0; i<this.mineList.length; i++) {
					var row = this.mineList[i]
					for (var j=0; j<row.length; j++) {
						var grid = row[j]
						grid.mineCount = this.roundMineCount(i, j)
					}
				}
			},
			roundMineCount: function (rowIndex, columnIndex) {
			    var mineCount = 0
				for (var i=rowIndex-1; i<=rowIndex+1; i++) {
				    if (i<0 || i>=this.mineList.length) {
				        continue
					}
				    for (var j=columnIndex-1; j<=columnIndex+1; j++) {
				        if (j<0 || j>=this.mineList[i].length || (i==rowIndex&&j==columnIndex)) {
				            continue
						}
						if (this.mineList[i][j].mine) {
							mineCount++
						}
					}
				}
				return mineCount
			},
			randomMine: function (exceptionRow, exceptionColumn, isLongpress) {
				var unMineList = []
				for (var row in this.mineList) {
				    for (var column in this.mineList[row]) {
				        if (isLongpress==false&&exceptionRow==row&&exceptionColumn==column) {
				            continue
						}
						unMineList.push(this.mineList[row][column])
					}
				}
				for (var i=0; i<this.mineCount; i++) {
					var randomIndex = Math.floor(Math.random()*unMineList.length)
					var grid = unMineList[randomIndex]
					grid.mine = true
					unMineList.splice(randomIndex, 1)
				}
			},
			resetGame: function () {
			    this.started = false
				this.gameStatus = 0
				this.exposeCount = 0
				for (var i=0; i<this.rowNumber; i++) {
				    var row
				    if (i<this.mineList.length) {
				        row = this.mineList[i]
						this.mineList.splice(i, 1, row)
					} else  {
				        row = []
						this.mineList.push(row)
					}
				    for (var j=0; j<this.columnNumber; j++) {
				        var grid
						if (j<row.length) {
				            grid = row[j]
						} else {
				            grid = {}
				            row.push(grid)
						}
						grid.status = 0
						grid.mine = false
						grid.mineCount = 0
					}
				}
			}
		},
        created: function () {
			this.resetGame()
        }
    }
	function contains(arr, obj) {
	    for (var i in arr) {
	        if (arr[i] === obj) {
	            return true
			}
		}
		return false
	}
	function toast(text, duration) {
		modal.toast({
			message: text,
			duration: duration
		})
	}
</script>

<style scoped>
	.content {
		flex-direction: column;
		background-color: #F5F5F5;
	}
	.content-header {
		flex: 1;
	}
	.content-footer {
		height: 60wx;
		flex-direction: row;
		justify-content: center;
		align-items: center;
	}
	.reset-button {
		font-size: 30wx;
		color: orange;
	}
	.row {
		flex-direction: row;
	}
	.grid {
		flex-direction: row;
		justify-content: center;
		align-items: center;
	}
	.grid-bg {
		position: absolute;
		left: 2wx;
		top: 2wx;
		right: 2wx;
		bottom: 2wx;
	}
	.grid-image {
		position: absolute;
		left: 2wx;
		top: 2wx;
		right: 2wx;
		bottom: 2wx;
	}
	.grid-text {
		font-size: 30px;
		color: white;
	}
</style>
