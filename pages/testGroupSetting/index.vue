<template>
	<view class="tgs-win">
		<!-- 小组列表 -->
		<view class="tgs-group-list">
			<test-group-list header></test-group-list>
			<radio-group
				@change="selectGroup">
				<view
					v-for="(group, idx) in groups"
					:key='idx'>
					<test-group-list
						:group='group'
						:selected='selectedIndex === idx'>
					</test-group-list>
				</view>
			</radio-group>
		</view>
		
		<!-- 删除小组按钮 -->
		<view class="tgs-delete-btn">
			<button
				size='mini'
				@click="openDeleteGroupPopup">
				删除小组
			</button>
		</view>
		<!-- 导出到excel按钮 -->
		<view class="tgs-export-btn">
			<view class="tgs-export-btn-flex">
				<!-- 轮次成绩 -->
				<!-- <view style="margin-bottom: 10rpx;">
					<button
						size='mini'
						@click="exportRoundScores">
						导出原始成绩(含轮次)
					</button>
				</view> -->
				<!-- 最优成绩 -->
				<view>
					<button
						size='mini'
						@click="exportFinalScores">
						导出成绩
					</button>
				</view>
			</view>
		</view>
		<!-- 新建小组按钮 -->
		<view class="tgs-create-btn">
			<button
				size='mini'
				@click="openCreateGroupPopup">
				新建小组
			</button>
		</view>
		<!-- 进入测试按钮 -->
		<view class="tgs-enter-test">
			<view>
				<text style="font-size: small;">
					当前小组：
				</text><br>
				<view style="margin: 10rpx 0;">
					<text style="color: #007AFF; font-size: large;">
						{{selectedGroupName}}
					</text>
				</view>
				<button
					@click="enterTest">
					进入测试
				</button>
			</view>
			
		</view>
		
		<!-- 新建小组的模态框 -->
		<uni-popup ref="createGroupPopup"
			type="dialog">
			<uni-popup-dialog type="info"
				mode='input'
				title="新建小组"
				placeholder="在此输入小组名称"
				@confirm="confirmCreateGroup"></uni-popup-dialog>
		</uni-popup>
		<!-- 删除小组的模态框 -->
		<uni-popup ref="deleteGroupPopup"
			type="dialog">
			<uni-popup-dialog
				title="删除小组？"
				@confirm="confirmDeleteGroup"></uni-popup-dialog>
		</uni-popup>
		
		<!-- 导出成功提示消息 -->
		<uni-popup ref='exportSuccessPopup'
			type='message'>
			<uni-popup-message
				:message="exportToastMsg"></uni-popup-message>
		</uni-popup>
		<!-- 导出失败提示消息 -->
		<uni-popup ref='exportFailPopup'
			type='message'>
			<uni-popup-message
				type='error'
				:message="exportToastMsg"></uni-popup-message>
		</uni-popup>
	</view>
</template>

<script>
	import TestGroupList from '../../components/TestGroupList.vue'
	import fitnessItemMap from '../../utils/fitnessItemMap.js'
	
	// import xlsx from 'xlsx'
	// const fileTools = uni.requireNativePlugin('replex-FileTools')
	const xlsModule = uni.requireNativePlugin("XM-XlsOutput")
	
	export default {
		components: {
			TestGroupList,
		},
		data() {
			return {
				fitnessItemName: '',//项目名
				fitnessItemNameZh: '',//项目名，中文
				fitnessItemCode: '',//项目编号
				fitnessItemPrefer: '',
				groups: [{
					groupId: '',
					groupName: '',
					createTime: '',
					exportTime: ''
				}],//写成这样才不会报错
				selectedIndex: 0,
				exportToastMsg: '',
			}
		},
		computed: {
			selectedGroupId() {
				return this.groups[this.selectedIndex].groupId
			},
			selectedGroupName() {
				return this.groups[this.selectedIndex].groupName
			},
			fileName() {
				return `${this.selectedGroupName}_${this.fitnessItemNameZh}`
			}
		},
		onUnload() {
			//返回时关闭数据库
			plus.sqlite.closeDatabase({
				name: 'data',
				success: () => {
					console.log('成功关闭数据库')
				},
				fail: () => {
					console.log('失败关闭数据库')
				},
			})
		},
		onLoad(args) {
			//从上个页面获取传入的参数，参数是项目名
			const { fitnessItemName } = args
			this.fitnessItemName = fitnessItemName
			this.fitnessItemNameZh = fitnessItemMap.get(fitnessItemName).nameZh
			this.fitnessItemCode = fitnessItemMap.get(fitnessItemName).code
			this.fitnessItemPrefer = fitnessItemMap.get(fitnessItemName).prefer
			this.openDbFetchGroups()
		},
		onReady() {
			//设置导航栏标题
			uni.setNavigationBarTitle({
				title: this.fitnessItemNameZh,
			})
		},
		methods: {
			//打开数据库，不存在则新建
			openDbFetchGroups() {
				plus.io.requestFileSystem(plus.io.PRIVATE_DOC, fs => {
					fs.root.getDirectory('db', {
						create: true
					}, entry => {
						const path = `${entry.fullPath}data.db`
						const isOpened = plus.sqlite.isOpenDatabase({
							name: 'data',
							path,
						})
						if (!isOpened) {//如果没打开
							plus.sqlite.openDatabase({
								name: 'data',
								path,
								success: () => {
									console.log('成功打开数据库')
									//创建groups和scores两个表
									plus.sqlite.executeSql({
										name: 'data',
										sql: [
											'create table if not exists groups("groupId" INTEGER PRIMARY KEY AUTOINCREMENT, "groupName" CHAR(30), "fitnessItemCode" CHAR(10), "createTime" CHAR(30), "exportTime" CHAR(30))',
											'create table if not exists scores("_id" INTEGER PRIMARY KEY AUTOINCREMENT, "groupId" INTEGER, "personId" CHAR(30), "score" CHAR(10), "round" CHAR(10), "testTime" CHAR(30))'
										],
										success: () => {
											console.log('成功创建表')
											plus.sqlite.selectSql({
												name: 'data',
												sql: `select groupId, groupName, createTime, exportTime from groups where fitnessItemCode = '${this.fitnessItemCode}'`,
												success: res => {
													//如果groups表内还没有小组，则初始化一条数据
													if (res.length === 0) {
														//在创建完表之后，groups表需要写入一条初始数据
														const groupName = '临时测试'
														const createTime = this.getNowTime()
														const exportTime = ''
														plus.sqlite.executeSql({
															name: 'data',
															sql: `insert into groups('groupName', 'fitnessItemCode', 'createTime', 'exportTime') values("${groupName}", "${this.fitnessItemCode}", "${createTime}", "${exportTime}")`,
															success: () => {
																console.log('成功初始化小组')
																this.fetchGroups()
															},
															fail: () => { console.log('失败初始化小组') }
														})
													}
													//如果已经有小组，则获取小组
													else {
														this.groups = res
														console.log('成功获取已有小组')
													}
												},
												fail: () => { console.log('失败查询小组') }
											})
										},
										fail: () => { console.log('失败创建表') }
									})
								},
								fail: () => { console.log('失败打开数据库') }
							})
						}
						else {
							//从groups表里获取全部行
							this.fetchGroups()
						}
					}, () => { console.log('失败新建db文件夹') })
				}, () => { console.log('失败打开文件系统') })
			},
			//导出到excel，最终成绩
			exportFinalScores() {
				plus.sqlite.selectSql({
					name: 'data',
					sql: `select personId, ${this.fitnessItemPrefer}(score) as finalScore, testTime from scores where groupId = ${this.selectedGroupId} group by personId`,
					success: res => {
						if (res.length > 0) {//至少有1条数据
							const dataJson = res.map(person => {
								const { personId, finalScore, testTime } = person
								const row = {}
								row['个人编号'] = personId
								row[`${this.fitnessItemNameZh}`] = finalScore
								row['测试时间'] = testTime
								return row
							})
							const ret = this.dataWriteToFile(dataJson)
							if (ret) {
								this.showExportToast('success')
							}
							else {
								this.showExportToast('fail', '导出失败！')
							}
						}
						else {
							this.showExportToast('fail', '导出失败：该小组还没有成绩！')
						}
					},
					fail: () => { console.log('失败获取小组成绩') }
				})
			},
			//导出到excel，轮次成绩
			// exportRoundScores() {
			// 	plus.sqlite.selectSql({
			// 		name: 'data',
			// 		sql: `select * from scores where groupId = ${this.selectedGroupId}`,
			// 		success: res => {
			// 			const dataJson = res.map(person => {
			// 				const { personId, score, round, testTime } = person
			// 				const row = {}
			// 				row['个人编号'] = personId
			// 				row[`${this.fitnessItemNameZh}`] = score
			// 				row['轮次'] = round
			// 				row['测试时间'] = testTime
			// 				return row
			// 			})
			// 			this.dataWriteToFile(dataJson, '原始成绩(含轮次)')
			// 		},
			// 		fail: () => { console.log('失败获取小组成绩') }
			// 	})
			// },
			// 使用“Android导出数据到Excel表”插件，这个方案轻巧
			dataWriteToFile(dataJson) {
				const directoryName = `-汇洋体测/${this.fitnessItemNameZh}/导出`//导出到的文件夹
				const sheetName = `${this.fitnessItemNameZh}`//sheet名，必需
				
				const rowHeader = [
					{
					cellList: Object.keys(dataJson[0]).map(item => ({ text: item }))//表头
					}
				]
				const rows = dataJson.map(person => ({
					cellList: Object.values(person).map(item => ({ text: item }))
				}))
				const rowList = rowHeader.concat(rows)
				
				const data = {
					directoryName,
					fileName: this.fileName,
					sheetList: [{
						sheetName,
						rowList
					}]
				}
				const ret = xlsModule.outPutExcel(data)
				return ret.success//该字段如果导出excel成功为true，失败为false
			},
			/*
			把json数据写入到excel文件
			使用xlsx模块和“文件处理集成工具”插件
			因为xlsx模块过大，此方法不使用
			*/
			// dataWriteToFile(dataJson, mode) {
			// 	//使用xlsx生成excel的base64
			// 	const sheet = xlsx.utils.json_to_sheet(dataJson)
			// 	const wb = xlsx.utils.book_new()
			// 	xlsx.utils.book_append_sheet(wb, sheet)
			// 	const base64Data = xlsx.write(wb, {
			// 		bookType: 'xlsx',
			// 		type: 'base64',
			// 	})
			// 	//使用fileTools插件把base64转成excel文件
			// 	const fileName = `${this.selectedGroupName}_${mode}_${this.fitnessItemNameZh}.xlsx`
			// 	fileTools.base64ToFile(base64Data, fileName, res => {
			// 		const oldPath = res.data
			// 		//打开fileTools保存的文件路径
			// 		plus.io.resolveLocalFileSystemURL(oldPath, entry => {
			// 			//打开一个项目专门保存excel的路径
			// 			const newDir = `/storage/emulated/0/-汇洋体测/${this.fitnessItemNameZh}/导出`
			// 			plus.io.resolveLocalFileSystemURL(newDir, entry2 => {
			// 				//newDir + '/' + fileName为目标文件的路径，create: false不新建
			// 				//先判断该文件是否存在，如果存在则先删除
			// 				entry2.getFile(newDir + '/' + fileName, { create: false }, entry3 => {
			// 					//已存在则先删除
			// 					entry3.remove(() => {
			// 						//拷贝excel文件
			// 						this.copyFileTo(entry, entry2, fileName)
			// 					}, () => { console.log('失败删除') })
			// 				}, () => {
			// 					console.log('失败打开')
			// 					//失败打开，说明目标文件还不存在，直接拷贝
			// 					this.copyFileTo(entry, entry2, fileName)
			// 				})
			// 			}, () => { console.log('失败') })
			// 		}, () => { console.log('失败') })
			// 	})
			// },
			/*
			plus.io的拷贝文件操作
			这是配套xlsx模块和“文件处理集成工具”插件而用的
			“文件处理集成工具”插件需要把生成的文件从原始位置移动到目标位置
			*/
			// copyFileTo(entry, entry2, fileName) {
			// 	entry.copyTo(entry2, fileName, () => {
			// 		//拷贝成功，删除原始文件
			// 		entry.remove(() => {
			// 			console.log('成功删除原始文件')
			// 		}, () => { console.log('失败删除原始文件') })
			// 		this.showExportToast(fileName)
			// 	}, err => {
			// 		console.log(err)//拷贝失败打印错误消息
			// 	})
			// },
			// 展示导出成功后的提示
			showExportToast(type, failMsg) {
				if (type === 'success') {
					//更新导出时间并刷新显示
					this.updateExportTime()
					//成功后的提示
					this.exportToastMsg = `成功导出到：/-汇洋体测/${this.fitnessItemNameZh}/导出/${this.fileName}.xls`
					this.$refs.exportSuccessPopup.open()
				}
				else if (type === 'fail') {
					this.exportToastMsg = failMsg
					this.$refs.exportFailPopup.open()
				}
				
			},
			//小组导出后更新exportTime字段
			updateExportTime() {
				const exportTime = this.getNowTime()
				plus.sqlite.executeSql({
					name: 'data',
					sql: `update groups set exportTime = '${exportTime}' where groupId = ${this.selectedGroupId}`,
					success: () => {
						console.log('成功更新小组')
						//刷新数据
						this.fetchGroups()
					},
					fail: () => { console.log('失败更新小组') }
				})
			},
			//从groups表里获取全部行
			fetchGroups() {
				return new Promise((resolve, reject) => {
					plus.sqlite.selectSql({
						name: 'data',
						sql: `select groupId, groupName, createTime, exportTime from groups where fitnessItemCode = '${this.fitnessItemCode}'`,
						success: res => {
							this.groups = res
							resolve(this.groups.length)
						},
						fail: () => {
							console.log('失败获取组名')
							reject(0)
						}
					})
				})
			},
			//使用radio选择小组
			selectGroup(evt) {
				let i = 0, sentry = true
				while (sentry) {
					if (this.groups[i].groupId === evt.detail.value) {
						sentry = false
					}
					else {
						i ++
					}
				}
				//所选小组在this.groups里的索引，用来高亮显示
				this.selectedIndex = i
			},
			//导航到测试页面
			enterTest() {
				uni.navigateTo({
					url: `../${this.fitnessItemName}/index?groupName=${this.selectedGroupName}&groupId=${this.selectedGroupId}`
				})
			},
			//打开删除小组的模态框
			openDeleteGroupPopup() {
				this.$refs.deleteGroupPopup.open()
			},
			//打开输入组名的模态框
			openCreateGroupPopup() {
				this.$refs.createGroupPopup.open()
			},
			//模态框确定删除小组
			confirmDeleteGroup(done) {
				plus.sqlite.selectSql({
					name: 'data',
					sql: `select count(*) as groupCount from groups where fitnessItemCode = '${this.fitnessItemCode}'`,
					success: res => {
						//如果该项小组个数大于1则删除，如果只剩1个小组则不删除
						if (res[0].groupCount > 1) {
							plus.sqlite.executeSql({
								name: 'data',
								sql: `delete from groups where groupId = ${this.selectedGroupId}`,
								success: () => {
									console.log('成功删除小组')
									//刷新数据
									this.fetchGroups()
									this.selectedIndex = 0//删除完之后把选中行设为第一行
								},
								fail: () => { console.log('失败删除小组') }
							})
						}
					},
					fail: () => { console.log('失败查询小组个数') }
				})
				done()
			},
			//模态框确定新建小组
			confirmCreateGroup(done, value) {
				//如果有输入，则保存到数据库
				const groupName = value.replace(/\s/g, '')//先剔除空格
				const len = groupName.length
				if (len > 0 && len <= 16) {//小组名称最大16个字符
					const createTime = this.getNowTime()
					const exportTime = ''
					plus.sqlite.executeSql({
						name: 'data',
						sql: `insert into groups('groupName', 'fitnessItemCode', 'createTime', 'exportTime') values("${groupName}", "${this.fitnessItemCode}", "${createTime}", "${exportTime}")`,
						success: async () => {
							console.log('成功新建小组')
							//刷新数据
							const ret = await this.fetchGroups().then(res => res).catch(err => err)
							if (ret) {
								this.selectedIndex = ret - 1//新建完之后选中新行
							}
						},
						fail: () => { console.log('失败新建小组') }
					})
				}
				done()
			},
			//获取当前时间
			getNowTime() {
				return new Date(Date.now() + 8 * 3600 * 1000).toISOString().slice(2, -8)
			},
		}
	}
</script>

<style>
	.tgs-win {
		padding-bottom: 30rpx;
	}
	.tgs-group-list {
		margin-left: 12rpx;
	}
	.tgs-delete-btn {
		position: fixed;
		top: 20rpx;
		right: 20rpx;
	}
	.tgs-export-btn {
		position: fixed;
		top: 80rpx;
		right: 20rpx;
	}
	.tgs-export-btn-flex {
		display: flex;
		flex-direction: column;
	}
	.tgs-create-btn {
		position: fixed;
		top: 140rpx;
		right: 20rpx;
	}
	.tgs-enter-test {
		position: fixed;
		bottom: 40rpx;
		right: 20rpx;
		width: 110rpx;
	}
</style>
