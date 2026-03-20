<template>
	<view class="container">
		<navbar :title="$t('bill')" :show-back="true" :fixed="true" fontSize="18px" :placeholder="true" />
			<view style="margin: 32rpx;">
			<tm-input v-model="pageState.key" :placeholder="$t('fillInPhoneNumber')" left-icon="search-2-line" iconColor="#B3C1D9" color="#fff" @input="handleInput" @clear="onSearch" />
			</view>
			<view class="dateSearch">
			<view class="date-picker-container">
					<picker mode="date" @change="bindDateChange" fields="month">
						<view class="uni-input" style=" color: #020204; font-size: 28rpx;">{{formatDateToYearMonth(pageState.date)}}
						<tm-icon name="arrow-down-s-fill" size="35rpx"></tm-icon>
						</view>
					</picker>
				<view>
				<text class="total-income-text">{{$t('totalIncome')}}：<text class="cui-text-price">
						{{monthCountAmouth}}</text></text>
						<!-- <text class="total-income-text">{{$t('totalIncome')}}：<text class="cui-text-price">
								0</text></text> -->
						</view>
			</view>
			</view>
		<scroll-box :sub-height="'400rpx'" :list-length="todayList.length || list.length" :can-refresher="false"
			@refresherrefresh="onRefresh" @scrolltolower="onScrollToLower">
			<template #list>
				<!-- 今日数据区域 -->
				<view class="teday" v-if="todayList.length > 0">
					<view class="date-title">
						<text class="date-text">{{ todayList[0].dateText }} {{$t('today')}}</text>
						<view>
						<!-- <text class="date-amount">出</text>
						<text class="todayAmount">0.00</text> -->
						<text class="date-amount">入</text>
						<text class="todayAmount">{{todayAmount}}</text>
							
						</view>
					</view>
					<view class="list" v-for="(item, index) in todayList" :key="item.ledgerId || index">
						<next-swipe-action :btnGroup="btnGroup" :index="index" @btnClick="onclick">
							<view class="item">
								<view class="item-left">
									<view class="item-tel">{{item.mobile}}</view>
									<text class="item-time">{{formatDateToHM(item.recordTime)}}</text>
								</view>
								<text class="item-amount">+{{item.amount}}</text>
							</view>
						</next-swipe-action>
					</view>
				</view>
				<!-- 昨日数据区域 -->
				<view class="teday" v-if="todayList.length > 0">
					<view class="date-title">
						<text class="date-text">{{ todayList[0].dateText }} 昨天</text>
						<view>
					<!-- 	<text class="date-amount">出</text>
						<text class="todayAmount">0.00</text> -->
						<text class="date-amount">入</text>
						<text class="todayAmount">{{todayAmount}}</text>
							
						</view>
					</view>
					<view class="list" v-for="(item, index) in todayList" :key="item.ledgerId || index">
						<next-swipe-action :btnGroup="btnGroup" :index="index" @btnClick="onclick">
							<view class="item">
								<view class="item-left">
									<view class="item-tel">{{item.mobile}}</view>
									<text class="item-time">{{formatDateToHM(item.recordTime)}}</text>
								</view>
								<text class="item-amount">+{{item.amount}}</text>
							</view>
						</next-swipe-action>
					</view>
				</view>
				<view class="monthTitle">
					<text style="color: #020204;font-size: 28rpx;">{{ formatDateToYearMonth(pageState.date) }}</text>
					<text style="color: #6E758D; margin-left: 15rpx; margin-right: 15rpx;">|</text>
					<text style="color: #6E758D;">{{$t('totalIncome')}} <text class="cui-text-price">{{monthCountAmouth}}
						</text></text>
					<!-- <text style="color: #6E758D; margin-left: 20rpx;">总出账<text class="cui-text-price">0.00
						</text></text> -->
				</view>
				<!-- 月份数据区域 -->
				<view class="teday" v-if="list.length > 0">
					<view class="list" v-for="(item, monthIndex) in list" :key="monthIndex">
						<view class="date-title">
							<text class="date-text">{{formatDateToMonthDay(item.date)}}</text>
							<view>
							<!-- <text class="date-amount">出</text>
							<text class="todayAmount">0.00</text> -->
							<text class="date-amount">入</text>
							<text class="todayAmount">{{item.totalAmount}}</text>	
							</view>
						</view>
						<next-swipe-action :btnGroup="btnGroup"
							@btnClick="(e) => delMonth(e, monthIndex, Number(dateIndex))"
							v-for="(itemData, dateIndex) in item.data" :key="dateIndex">
							<view class="item">
								<view class="item-left">
									<view class="item-tel">{{itemData.mobile}}</view>
									<text class="item-time">{{formatDateToHM(itemData.recordTime)}}</text>
								</view>
								<text class="item-amount">+{{itemData.amount}}</text>
							</view>
						</next-swipe-action>
					</view>
				</view>
			</template>
		</scroll-box>
		<open-ledger-icon @open="openLedger" style="z-index: 999;"></open-ledger-icon>
		<ledger-new :show="isShow" @close="closeLedger" :buttonShow="true" @success="handleLedgerSuccess"
			:billId="billId"></ledger-new>
	</view>
</template>

<script setup lang="ts">
	import { ref, nextTick, watch } from "vue"
	import { getTodyLedger, delLedger, getLedgerPage, getMonthAmount } from "@/request-api/api"
	import { onLoad } from '@dcloudio/uni-app';
	import scrollBox from "@/components/scroll-list-box/scroll-box.vue"
	import { useI18n } from 'vue-i18n';
	const { t, locale } = useI18n();

	const todayAmount = ref<number>(0);
	const todayList = ref<any[]>([]);
	const monthCountAmouth = ref("")
	//传参
	const billId = ref("")
	const pageState = ref({
		page: 1,
		limit: 10,
		key: '',
		date: ''
	})

	const list = ref<any[]>([])
	const isLoading = ref(false)
	const hasMore = ref(true)

	const btnGroup = ref([
		{ name: t('edit'), action: 'edit', style: { bgColor: '#f9ae3d' } },
		{ name: t('delete'), action: 'del', style: { bgColor: '#ff4d4f' } }
	]);

	// 输入框输入事件处理
	function handleInput() {
		nextTick(() => {
			onSearch()
		})
	}

	// 初始化数据 
	function loadData(pageNum : number = 1) : void {
		if (isLoading.value) return
		isLoading.value = true
		const params = {
			page: pageNum,
			limit: pageState.value.limit,
			key: pageState.value.key,
			date: pageState.value.date
		}

		getLedgerPage(params).then((res : any) => {
			if (pageNum === 1) {
				// 刷新，清空旧数据
				list.value = groupDataByDate(res.list) || []
				pageState.value.page = 1
			} else {
				// 加载更多
				list.value.push(...(groupDataByDate(res.list) || []))
				pageState.value.page = pageNum
			}

			// 判断是否还有更多数据
			hasMore.value = (res.list || []).length === pageState.value.limit
		}).catch((err : any) => {
			console.error('数据加载失败', err)
			uni.showToast({ title: t('failedToLoad'), icon: 'none' })
		}).finally(() => {
			isLoading.value = false
		})
	}

	// 搜索方法 - 回车搜索
	function onSearch() : void {
		// 重置分页，从第一页开始
		pageState.value.page = 1
		hasMore.value = true
		getTodyLedgerData()
		getMonthAmountData()
		loadData(1)
	}

	// 日期选择器变化事件
	function bindDateChange(e : any) {
		pageState.value.page = 1
		// 使用月份选择器的值，添加固定日期部分用于查询
		pageState.value.date = e.detail.value + "-15 00:00:00"
		hasMore.value = true
		getMonthAmountData()
		loadData(1)
	}

	// 下拉刷新事件
	function onRefresh() : void {
		loadData(1)
		getTodyLedgerData()
	}

	// 刷新当前页
	function refreshCurrentPage() : void {
		loadData(pageState.value.page)
	}

	// 只更新金额数据，不重置分页
	function refreshAmounts() : void {
		getTodyLedgerData()
		getMonthAmountData()
	}

	// 触底划入事件 - 加载下一页
	function onScrollToLower() : void {
		if (isLoading.value) return;
		if (!hasMore.value) {
			uni.showToast({ title: t('lastPage'), icon: 'none' })
			return
		}
		loadData(pageState.value.page + 1)
	}
	const setPageLocale = () => {
		//判断当前语言，将内置组件国际化
		if (locale.value === 'en') {
			uni.setLocale('en');
		} else if (locale.value === 'zh') {
			uni.setLocale('zh');
		}
	};
	watch(locale, () => {
		setPageLocale();
	}, { immediate: true });

	onLoad(() => {
		getCurrentDateZeroTime()
		init()
		loadData(1)
		// 在页面加载时注册监听
		uni.$on('refreshBlack', () => {
			setPageLocale();
			refreshCurrentPage()
		})
	})
	// 数据请求
	const init = async () => {
		await getTodyLedgerData();
		await getMonthAmountData();
	};
	//获取今天数据
	const getTodyLedgerData = async () => {
		let tel = pageState.value.key
		const res = await getTodyLedger(tel);
		todayAmount.value = calculateTotalAmount(res) || 0
		console.log(res,"今天数据")
		todayList.value = formatDateList(res) || []
	};
	//获取月份金额
	const getMonthAmountData = async () => {
		let date = pageState.value.date
		const res = await getMonthAmount(date);
		monthCountAmouth.value = res || 0
	};

	//编辑删除当天数据
	const onclick = async (e : any) => {
		const currentItem = todayList.value[e.index];
		if (e.item.action === 'del') {
			const res = await uni.showModal({
				title: t('deleteconfirmation'),
				content: t('doYouWantToDeleteThisRecord'),
				confirmText: t('yes'),
				cancelText: t('cancel')
			});
			if (res.confirm) {
				try {
					await delLedger(currentItem.billId);
					todayList.value.splice(e.index, 1);
					// 更新今日数据和金额，不重置分页（今日数据不在分页列表中）
					refreshAmounts();
					uni.showToast({ title: t('success'), icon: "success" });
				} catch (err) {
					uni.showToast({ title: t('deletionFailed'), icon: "none" });
				}
			}
		} else {
			billId.value = currentItem.billId
			isShow.value = true;
		}
	};
	//编辑删除当月数据
	const delMonth = async (e, monthIndex : number, dateIndex : number) => {

		const outerItem = list.value[monthIndex];
		const currentItemData = outerItem.data[dateIndex];
		if (e.item.action === 'del') {
			const res = await uni.showModal({
				title: t('deleteconfirmation'),
				content: t('doYouWantToDeleteThisRecord'),
				confirmText: t('yes'),
				cancelText: t('cancel')
			});
			if (!res.confirm) return;

			try {
				await delLedger(currentItemData.billId);
				outerItem.data.splice(dateIndex, 1);
				if (outerItem.data.length === 0) {
					list.value.splice(monthIndex, 1);
				}
				// 刷新当前页数据和金额，不重置分页
				refreshCurrentPage();
				refreshAmounts();
				uni.showToast({ title: t('success'), icon: "success" });
			} catch (err) {
				console.error("月份账单删除失败：", err);
				uni.showToast({ title: t('deletionFailed'), icon: "none" });
			}
		} else {
			billId.value = currentItemData.billId
			isShow.value = true;
		}
	};
	//计算今天的总收入
	const calculateTotalAmount = (arr : any[], decimal : number = 2) : number => {
		let total = 0;
		arr.forEach((item, index) => {
			if (!item || !('amount' in item)) {
				return;
			}
			const amount = Number(item.amount);
			if (!isNaN(amount)) {
				total += amount;
			} else {
				console.warn(`第${index + 1}个元素的amount不是有效数值：`, item.amount);
			}
		});
		const result = Number(total.toFixed(decimal));
		return result;
	};

	//需要日期格式
	const formatDateList = (list: any[]): any[] => {
	    // 校验入参是否为有效数组
	    if (!Array.isArray(list) || list.length === 0) {
	        return [];
	    }
	
	    return list.map(item => {
	        const newItem = { ...item };
	
	        if (newItem.recordTime) {
	            try {
	                // 拆分日期部分和时间部分（按空格分割）
	                const [datePart, timePart] = newItem.recordTime.split(' ');
	                if (datePart && timePart) {
	                    // 处理日期部分：2026-01-12 -> 拆分出 年、月、日
	                    const [year, month, day] = datePart.split('-');
	                    // 生成日期文本：月-日（如 01-12）
	                    newItem.dateText = `${month}-${day}`;
	                    
	                    // 处理时间部分：14:41:17 -> 拆分出 时、分、秒，只取时分
	                    const [hour, minute] = timePart.split(':');
	                    // 生成时间文本：时：分（中文冒号，如 14：41）
	                    newItem.timeText = `${hour}：${minute}`;
	                }
	            } catch (error) {
	                console.error('日期格式化失败：', error);
	                newItem.dateText = '';
	                newItem.timeText = '';
	            }
	        } else {
	            newItem.dateText = '';
	            newItem.timeText = '';
	        }
	        return newItem;
	    });
	};
	//获取当前时间
	function getCurrentDateZeroTime() {
		const now = new Date();
		// 获取年、月、日并补零
		const year = now.getFullYear();
		const month = String(now.getMonth() + 1).padStart(2, '0');
		const day = String(now.getDate()).padStart(2, '0');
		// 固定时分秒为00:00:00
		pageState.value.date = `${year}-${month}-${day} 00:00:00`;
	}
	//处理分页数据
	function groupDataByDate(dataList : any[]) : any[] {
		// 判空处理
		if (!Array.isArray(dataList) || dataList.length === 0) {
			return [];
		}

		const groupObj = {};

		// 遍历原始数组，进行分组归类
		dataList.forEach(item => {
			// 提取仅包含年月日的纯日期作为分组关键字
			let dateKey = '未知日期';
			if (item.recordTime) {
				try {
					// 将item.recordTime转为Date对象
					const dateObj = new Date(item.recordTime);
					// 验证Date对象有效性
					if (!isNaN(dateObj.getTime())) {
						// 提取年月日，格式化为 YYYY-MM-DD 
						dateKey = dateObj.toISOString().split('T')[0];
					}
				} catch (e) {
					// 异常情况
					console.warn('无效的日期格式，已归类为"未知日期"：', item.recordTime);
				}
			}

			// 初始化分组数组并推入当前数据项
			if (!groupObj[dateKey]) {
				groupObj[dateKey] = [];
			}
			groupObj[dateKey].push(item);
		});

		// 转换对象为目标数组格式，计算当日总额并格式化金额
		const resultArray = Object.keys(groupObj).map(dateKey => {
			const dailyData = groupObj[dateKey];
			const dailyTotalAmount = dailyData.reduce((total, currentItem) => {
				const amount = Number(currentItem.amount) || 0;
				return total + amount;
			}, 0);

			const formattedTotalAmount = dailyTotalAmount % 1 === 0
				? String(dailyTotalAmount)
				: Number(dailyTotalAmount.toFixed(2)).toString();

			return {
				date: dateKey, // 分组日期
				data: dailyData, // 该日期对应的所有原始数据
				totalAmount: formattedTotalAmount // 格式化后的当日总额
			};
		});

		return resultArray;
	}
	//格式化日期
	const formatDateToHM = (dateStr : string) : string => {
		if (!dateStr || typeof dateStr !== 'string') return ''
		const standardStr = dateStr.replace(/：/g, ':').trim()
		const timePart = standardStr.split(' ')[1]
		if (!timePart) return ''
		const timeArr = timePart.split(':')
		if (timeArr.length < 2) return ''

		const padZero = (num : string | number) => {
			const n = Number(num)
			return isNaN(n) ? '00' : (n < 10 ? `0${n}` : `${n}`)
		}
		const hours = padZero(timeArr[0])
		const minutes = padZero(timeArr[1])
		return `${hours}:${minutes}`
	}
	//月份数据
	function formatDateToMonthDay(dateStr : string) : string {
		// 入参判空与类型校验，避免无效输入
		if (!dateStr || typeof dateStr !== 'string') {
			return '';
		}
		// 同时截取日期部分
		const pureDateStr = dateStr.replace(/-/g, '/').split(' ')[0];
		// 创建 Date 实例
		const date = new Date(pureDateStr);

		// 校验 Date 实例有效性
		if (isNaN(date.getTime())) {
			return '';
		}
		// 提取月份和日期，并进行补零处理
		const month = String(date.getMonth() + 1).padStart(2, '0');
		const day = String(date.getDate()).padStart(2, '0');

		// 拼接为补零后的 MM-DD 格式
		return `${month}-${day}`;
	}
	//格式年月
	function formatDateToYearMonth(dateStr : string) : string {
		// 入参判空与类型校验，避免无效输入
		if (!dateStr || typeof dateStr !== 'string') {
			return '';
		}
		// 同时截取日期部分（与参考方法逻辑一致，先处理字符串提取纯日期）
		const pureDateStr = dateStr.replace(/-/g, '/').split(' ')[0];
		// 创建 Date 实例
		const date = new Date(pureDateStr);

		// 校验 Date 实例有效性（避免无效日期格式导致的异常）
		if (isNaN(date.getTime())) {
			return '';
		}
		// 提取年份（无需补零）和月份（需补零处理，与参考方法保持一致）
		const year = date.getFullYear();
		const month = String(date.getMonth() + 1).padStart(2, '0');

		// 拼接为补零后的 YYYY-MM 格式
		return `${year}-${month}`;
	}
	//去记账
	const isShow = ref(false);
	// 打开弹窗
	const openLedger = () => {
		billId.value = ""
		nextTick(() => {
			isShow.value = true;
		});
	};

	const closeLedger = () => {
		isShow.value = false;
		billId.value = "";
	};

	const handleLedgerSuccess = async () => {
		pageState.value.page = 1
		hasMore.value = true
		init()
		loadData(1)
		uni.showToast({ title: t('success'), icon: "success", duration: 1500 });
	};
</script>

<style>
	/* .search-bar {
		padding: 25rpx;
		background-color: white;
		border-radius: 0 0 28rpx 28rpx;
		box-shadow: 2px 2px 8px rgba(0, 0, 0, 0.3);
		height: fit-content;
		overflow: hidden;
		touch-action: none;
		-webkit-touch-callout: none;
	} */
	.dateSearch{
		display: flex;
		height: 104rpx;
		width: 100%;
		/* background-color: #fff; */
		 align-items: center;
		 background-image:url(@/static/images/page/icon/bill-date-icon.png);
		 background-size: cover; 
		 background-position: center;
	}
	.item-box {
		background-color: white;
		border-radius: 28rpx;
		padding: 32rpx;
		margin-bottom: 25rpx;
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.date-picker-container {
		display: flex;
		margin: 25rpx;
		gap: 15rpx;
		justify-content: space-between;
		align-items: flex-start;
		height: fit-content;
		overflow: hidden;
		
	}

	.total-income-text {
		/* height: 80rpx; */
		/* display: flex; */
		align-items: center;
		padding: 0 20rpx;
		/* font-weight: 600; */
		color: #6E758D;
		white-space: nowrap;
		overflow: hidden;
		font-size: 28rpx;
	}

	.search-texe {
		font-size: 35rpx;
		font-weight: 600;
		margin-left: 8rpx;
		margin-top: 25rpx;
		margin-bottom: 25rpx;
	}

	.teday {
		margin: 32rpx;
		padding: 3rpx;
		background-color: white;
		border-radius: 28rpx;
	}

	.list {
		width: 100%;
		margin-bottom: 10rpx;
	}

	.item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 10rpx 0;
		margin: 10rpx 32rpx;
	}

	.item-left {
		display: flex;
		flex-direction: column;
	}

	.item-tel {
		font-size: 28rpx;
		color: #333;
	}

	.item-time {
		font-size: 24rpx;
		color: #999;
	}

	.item-amount {
		font-size: 32rpx;
		color: #FFAD52;
		margin-right: 10rpx;
	}

	.date-title {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 15rpx;
		/* font-weight: 600; */
		background: #F8F8FF;
		border-radius: 28rpx 28rpx 0 0;
		height: 130rpx;
		width:100%;
		
	}

	.date-text {
		font-size: 32rpx;
		color: #020204;
		margin: 44rpx 28rpx;
	}

	.date-amount {
		background: #EAEAF4 ;
		font-size: 28rpx;
		color: #6E758D;
		margin: 0 10rpx;
		padding: 5rpx;
		border-radius: 8rpx 8rpx 8rpx 8rpx;
	}
	.todayAmount{
		color: #020204;
		margin-right: 28rpx;
		font-size: 32rpx;
	}
	.monthTitle{
		width: 100%;
		height: 80rpx;
		border-radius: 8rpx;
		display: flex;
		align-items: center;
		margin: 32rpx;
	}

	.images {
		width: 60rpx;
		height: 60rpx;
		object-fit: contain;
	}

	.text {
		margin-top: 10rpx;
		font-size: 28rpx;
		color: #7933ff;
	}

	page {
		overflow: hidden;
		height: 100vh;
		background: #F8F8FF;
	}

	.container {
		width: 100%;
		height: 100vh;
		overflow: hidden;
		position: relative;
		background-color: #F8F8FF;
	}
</style>