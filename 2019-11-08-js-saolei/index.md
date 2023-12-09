# JS 扫雷小游戏


扫雷是一款大众类的益智小游戏,于 1992 年发行。游戏目标是在最短的时间内根据点击格子出现的数字找出所有非雷格子,同时避免踩雷,踩到一个雷即全盘皆输。

![JS实现的扫雷小游戏](/img/js-minesweeper.jpg "JS实现的扫雷小游戏")

## 游戏规则

在写扫雷之前，我们先了解下它的游戏规则：

- 扫雷是一个矩阵，地雷随机分布在其中的方格里。
- 方格上的数字代表着这个方格所在的九宫格内有多少个地雷。
- 游戏玩家用鼠标左键打开方格，用鼠标右键标记地雷。
- 踩到地雷，游戏失败。打开所有非雷方格，游戏胜利。

## 功能实现

### 矩阵的生成

用 html 中的表格 table，span 生成矩阵方格。把矩阵方格放入二维数组中，然后对单个方格添加鼠标事件。

```js
//初始化矩阵 (row-行数 col-列数)
function init_grid() {
  //生成矩阵html <tr>--行标签 <td>--列标签
  let gridHtml = "";
  for (let i = 0; i < row; i++) {
    gridHtml += "<tr>";
    for (let j = 0; j < col; j++) {
      gridHtml +=
        '<td><span class="blocks" onmousedown="block_click(' +
        i +
        "," +
        j +
        ',event)"></span></td>';
    }
    gridHtml += "<tr>";
  }
  //写入html
  document.getElementById("grid").innerHTML = gridHtml;

  //返回矩阵二维数组
  let blocks = document.getElementsByClassName("blocks");
  let grid = new Array();
  for (let i = 0; i < blocks.length; i++) {
    if (i % col === 0) {
      grid.push(new Array());
    }
    //初始化计雷数
    blocks[i].count = 0;
    grid[parseInt(i / col)].push(blocks[i]);
  }
  return grid;
}
```

### 方格打开与标记

通过 onmousedown 事件，传入点击的方格的坐标及 event，判断 event 为左键还是右键。左键打开方格，右键标记方格。

```js
//方格点击事件 _i：坐标i _j:坐标j e:鼠标事件
function block_click(_i, _j, e) {

    //跳过已打开的方格
    if (grid[_i][_j].isOpen) {
        return;
    }

    //鼠标左键打开方格
    if (e.button === 0) {
        ...
        //执行递归打开方格函数
        block_open(_i, _j);
    }
    //鼠标右键标记方格
    else if (e.button === 2) {
        let block = grid[_i][_j];
        if (block.innerHTML !== '▲') {
            block.innerHTML = '▲';
        } else {
            block.innerHTML = '';
        }
    }
}
```

### 地雷随机分布

- 第一次打开的方格不能为地雷，把生成地雷的函数放在第一次点击方格后。
- 通过循环用 Math.random() 函数来随机生成地雷的二维坐标。
- 判断坐标是否不为第一次点击方格的坐标以及该坐标没有雷存在。
  - 是则将方格设置为地雷，当前地雷数+1，将该方格所在九宫格内的方格的计雷数+1。
  - 否则跳过进入下个循环，直到地雷的数量达到设定的最大雷数，结束循环。

```js
//方格点击事件 _i：坐标i _j:坐标j e:鼠标事件
function block_click(_i, _j, e) {
  //跳过已打开的方格
  if (grid[_i][_j].isOpen) {
    return;
  }

  //鼠标左键打开方格
  if (e.button === 0) {
    //第一次打开
    if (isFirstOpen) {
      isFirstOpen = false;
      let count = 0; //当前地雷数
      //生成地雷
      while (count < maxCount) {
        //生成随机坐标
        let ri = Math.floor(Math.random() * row);
        let rj = Math.floor(Math.random() * col);

        //坐标不等于第一次点击方格的坐标 && 非雷方格
        if (!(ri === _i && rj === _j) && !grid[ri][rj].isMine) {
          grid[ri][rj].isMine = true; //自定义属性isMine代表方格为地雷
          count++; //当前地雷数+1
          //更新九宫格内非雷方格的计雷数
          for (let i = ri - 1; i < ri + 2; i++) {
            for (let j = rj - 1; j < rj + 2; j++) {
              //判断坐标防越界
              if (i > -1 && j > -1 && i < row && j < col) {
                //计雷数+1
                grid[i][j].count++;
              }
            }
          }
        }
      }
    }

    //执行打开方格函数
    block_open(_i, _j);
  }
}
```

### 递归打开方格

当打开的方格为计雷数为 0 的方格，自动打开九宫格内的非雷方格。如果打开的非雷方格九宫格内仍有非雷方格，用递归继续打开九宫格内的非雷方格，直到没有为止。

```js

//递归打开方格函数
function block_open(_i, _j) {

    let block = grid[_i][_j];
    op(block);

    //设定打开方格的状态与样式
    function op(block) {
        block.isOpen = true; //isOpen为自定义属性，设置为true代表已打开
        block.style.background = '#ccc'; //将背景设置为灰色
        block.style.cursor = 'default'; //将鼠标停留样式设置为默认
    }
    //打开计雷数为0的方格
    if (block.count === 0) {
        //遍历九宫格内的方格
        for (let i = _i - 1; i < _i + 2; i++) {
            for (let j = _j - 1; j < _j + 2; j++) {
                //判断是否越界&&跳过已打开的方格&&非雷
                if (i > -1 && j > -1 && i < row && j < col && !grid[i][j].isOpen && !grid[i][j].ismine) {
                    //递归打开方格函数
                    block_open(i, j);
                }
            }
        }
    }  // 踩雷
    else if (block.isMine) {
        ...
    }  //打开计雷数不为0的方格
    else {
        block.innerHTML = block.count; //显示计雷数
    }
}
```

### 踩雷游戏结束

打开方格为地雷时，提示游戏结束。通过遍历矩阵打开所有埋地雷位置。

```js

else if (block.isMine) {
	block.innerHTML = '雷'; //显示为 '雷'
	//遍历矩阵打开所有埋地雷的方格
	for (let i = 0; i < row; i++) {
		for (let j = 0; j < col; j++) {
			//找到地雷
			block = grid[i][j];
			if (!block.isOpen && block.isMine) {
				op(block); //设置打开状态和样式
				block.innerHTML = '雷'; //显示为 '雷'
			}
		}
	}
	clearInterval(timer); //游戏结束停止计时，清除定时器
	//提示游戏结束
	alert("你踩到雷了！游戏结束");
}

```

### 游戏胜利条件

当所有非雷方格被打开即为游戏胜利。在每次打开方格函数中都遍历一遍矩阵，当找到有未打开的非雷方格时则退出遍历，遍历完所有方格均未找到未打开的非雷方格时则游戏胜利。

```js
//方块点击事件 _i：坐标i _j:坐标j e:鼠标事件
function block_click(_i, _j, e) {
  //跳过已打开的方块
  if (grid[_i][_j].isOpen) {
    //...
  }
  //鼠标左键打开方块
  if (e.button === 0) {
    //...
  }
  //鼠标右键标记方块
  else if (e.button === 2) {
    //...
  }

  //遍历矩阵
  let isWin = true;
  for (let i = 0; i < row; i++) {
    for (let j = 0; j < col; j++) {
      let block = grid[i][j];

      //如果有未打开的非雷方块
      if (!block.isMine && !block.isOpen) {
        isWin = false;
      }
    }
  }
  if (isWin) {
    alert("游戏胜利");
  }
}
```

游戏逻辑部分到这里就结束了，剩余雷数和计时可用全局变量实现。

附：[源码地址](http://jsrun.net/XdXKp/edit)

**参阅资料**

- [原生 JS 实现扫雷 (分析+代码实现)](https://www.cnblogs.com/kilomChou/p/10652405.html)

