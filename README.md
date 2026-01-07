package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
	"sync"
	"time"
)

type Goods struct {
	Name      string
	Price     float64
	Inventory int
}

var GoodsMap = make(map[string]*Goods)
var m sync.Mutex

func teller(id int, orderIDChan <-chan int, goodsNameChan <-chan string, quanTityChan <-chan int, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		orderID, ok1 := <-orderIDChan
		goodsName, ok2 := <-goodsNameChan
		quanTity, ok3 := <-quanTityChan
		if !ok1 || !ok2 || !ok3 {
			break

		}
		//多个柜员对商品进行操作，要加锁
		m.Lock()
		goods, ok := GoodsMap[goodsName]
		if !ok {
			fmt.Printf("[柜员%d]订单%d失败，%s商品不存在\n", id, orderID, goodsName)
			m.Unlock()
			time.Sleep(1 * time.Second)
			continue
		}
		if goods.Inventory >= quanTity {
			goods.Inventory -= quanTity
			total := float64(quanTity) * goods.Price
			fmt.Printf("\n[柜员%d] 订单%d成功,商品%s,售出%d,总价钱%.2f,剩余库存%d", id, orderID, goodsName, quanTity, total, goods.Inventory)
		} else {
			total := float64(goods.Inventory) * goods.Price
			fmt.Printf("\n[柜员%d]订单%d失败，%s库存不足,剩于库存%d,卖出的总价%.2f\n", id, orderID, goodsName, goods.Inventory, total)
			goods.Inventory = 0
		}
		m.Unlock()
		time.Sleep(1 * time.Second)
	}
}

func main() {

	Scanner := bufio.NewScanner(os.Stdin)
	fmt.Println("请输入商品信息（货物名字，价格，剩余容量）输入exit结束")
	for Scanner.Scan() {
		liem := Scanner.Text()
		if liem == "exit" {
			break
		}
		Parts := strings.Fields(liem)
		if len(Parts) != 3 {
			fmt.Println("输入格式错误")
			continue
		}
		price, err := strconv.ParseFloat(Parts[1], 64)
		if err != nil {
			fmt.Println("价格不是数字")
			continue
		}
		inventory, err := strconv.Atoi(Parts[2])
		if err != nil {
			fmt.Println("库存不是整数")
			continue
		}
		GoodsMap[Parts[0]] = &Goods{
			Name:      Parts[0],
			Price:     price,
			Inventory: inventory,
		}
		fmt.Printf("已经存入商品%s,单价%.2f,库存%d\n", Parts[0], price, inventory)
	}
	//启动协程
	orderIDChan := make(chan int, 5)      //订单号
	goodsNameChan := make(chan string, 5) //订单名
	quanTityChan := make(chan int, 5)     //订单数量
	var wg sync.WaitGroup
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go teller(i, orderIDChan, goodsNameChan, quanTityChan, &wg)
	}
	fmt.Println("开始处理订单")
	goodsNames := make([]string, 0)
	for name := range GoodsMap {
		goodsNames = append(goodsNames, name)
	}
	if len(goodsNames) == 0 {
		fmt.Println("没商品可卖")
		return
	}
	for i := 1; i <= 10; i++ {
		goodsName := goodsNames[i%len(goodsNames)] // 轮询选择商品
		orderIDChan <- i
		goodsNameChan <- goodsName
		quanTityChan <- 2 //每次售出个数
	}
	close(orderIDChan)
	close(goodsNameChan)
	close(quanTityChan)

	wg.Wait()
	fmt.Println("\n订单处理完毕")
}
