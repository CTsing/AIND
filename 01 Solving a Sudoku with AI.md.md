**本节学习如何用AI解决数独问题，解决问题的核心思路：将复杂问题拆分成各个简单的问题，通过解决各个简单问题，最终是复杂问题得到解决。**


## 数独问题的求解思路：
### 1.问题描述
解决一个问题，需要先搞清楚该问题的内容，包括问题的描述，解决该问题的目的。百度百科中对数独的定义是
>“数独盘面是个九宫，每一宫又分为九个小格。在这八十一格中给出一定的已知数字和解题条件，利用逻辑和推理，在其他的空格上填入1-9的数字。使1-9每个数字在每一行、每一列和每一宫中都只出现一次，所以又称“九宫格”。”

所以，数独问题就是一个填数字的游戏。

### 2.问题抽象：如何用计算机语言定义数独问题
数独问题需要在一个9*9的格子中完成，所以第一件要做的是就是定义这个格子。
这里用1-9表示列号，A-I表示行号，在python中，可以用列表来存放行号与列号

```
cols='123456789'
rows='ABCDEFGHI'
```

定义每一个格子为一个box，用A1、A2来表示。这里构建一个cross(rows,cols)，通过传入行与列生成box

```
def cross(a, b):
      return [s+t for s in a for t in b]

boxes = cross(rows, cols)
boxes =
    ['A1', 'A2', 'A3', 'A4', 'A5', 'A6', 'A7', 'A8', 'A9',
     'B1', 'B2', 'B3', 'B4', 'B5', 'B6', 'B7', 'B8', 'B9',
     'C1', 'C2', 'C3', 'C4', 'C5', 'C6', 'C7', 'C8', 'C9',
     'D1', 'D2', 'D3', 'D4', 'D5', 'D6', 'D7', 'D8', 'D9',
     'E1', 'E2', 'E3', 'E4', 'E5', 'E6', 'E7', 'E8', 'E9',
     'F1', 'F2', 'F3', 'F4', 'F5', 'F6', 'F7', 'F8', 'F9',
     'G1', 'G2', 'G3', 'G4', 'G5', 'G6', 'G7', 'G8', 'G9',
     'H1', 'H2', 'H3', 'H4', 'H5', 'H6', 'H7', 'H8', 'H9',
     'I1', 'I2', 'I3', 'I4', 'I5', 'I6', 'I7', 'I8', 'I9']
```

接下来，定义units为一行、一列、一个3*3的方格，在这三个units中，1-9只能出现一次；定义peers为某一格子在其units中除其自身的其他格子，如A1在A行（第一行）中的peers为A2、A3、A4、A5、A6、A7、A8、A9。在python中，可用列表来存放units和peers。
```
row_units = [cross(r, cols) for r in rows]
# Element example:
# row_units[0] = ['A1', 'A2', 'A3', 'A4', 'A5', 'A6', 'A7', 'A8', 'A9']
# This is the top most row.

column_units = [cross(rows, c) for c in cols]
# Element example:
# column_units[0] = ['A1', 'B1', 'C1', 'D1', 'E1', 'F1', 'G1', 'H1', 'I1']
# This is the left most column.

square_units = [cross(rs, cs) for rs in ('ABC','DEF','GHI') for cs in ('123','456','789')]
# Element example:
# square_units[0] = ['A1', 'A2', 'A3', 'B1', 'B2', 'B3', 'C1', 'C2', 'C3']
# This is the top left square.

unitlist = row_units + column_units + square_units
units = dict((s, [u for u in unitlist if s in u]) for s in boxes)
peers = dict((s, set(sum(units[s],[]))-set([s])) for s in boxes)
```

现在数独问题的基本框架已定义好。我们定义问题的输入——以字符串的形式传入数独问题。例如下图所示的数独问题，我们用列表'..3.2.6..9..3.5..1..18.64....81.29..7.......8..67.82....26.95..8..2.3..9..5.1.3..'来表示（对于待填值，用'.'表示）。

![](http://i.imgur.com/d6maeIk.jpg)


接下来，需要定义一个grid_values()函数，将box与其对应的值（value）对应起来，我们用字典来表示box与value的关系:{box:value}。同时，我们需要将待填box的.替换成possible value（'123456789'）。
```
from utils import *

def grid_values(grid):
    """Convert grid string into {<box>: <value>} dict with '123456789' value for empties.

    Args:
        grid: Sudoku grid in string form, 81 characters long
    Returns:
        Sudoku grid in dictionary form:
        - keys: Box labels, e.g. 'A1'
        - values: Value in corresponding box, e.g. '8', or '123456789' if it is empty.
    """
    values = []
    all_digits = '123456789'
    for c in grid:
        if c == '.':
            values.append(all_digits)
        elif c in all_digits:
            values.append(c)
    assert len(values) == 81
    return dict(zip(boxes, values))
    return dict(zip(boxes, grid))
```

### 3.解决问题
#### 3.1 Constraint Propagation
经过前面两步，我们已经将一个数独问题构造为一个计算机能理解的结构。现在一个首要问题是——如何求解？这里引入一个概念：Constraint Propagation。
Constraint Propagation是AI中一个非常常用而且也效果显著的思路。Constraint Propagation通过使用所有在搜索空间中的约束条件，进而缩小搜索空间的范围。在数独问题中，boxes中的value就是我们的搜索空间，我们需要逐步减少待填格子的可填值（possible value）。在每一个units中，各个box的值不能重复，这是数独问题的constrain。下面我们通过两个策略来实现数独问题的Constraint Propagation。
- eliminate:将待填box中的.替换为'123456789'，根据其peers的取值情况，移除其peers包含的value。如图一所示，B5所在units所包含的值有1、2、3、5、6、8、9，所有B5的possible value为4和7。同理，其他box也通过这种方式处理，处理完后的数独如图二所示。
图一
![](http://i.imgur.com/YFTZzym.jpg)
图二
![](http://i.imgur.com/ne5anv4.jpg)
python实现eliminate代码如下：

```
from utils import *

def eliminate(values):
    """Eliminate values from peers of each box with a single value.

    Go through all the boxes, and whenever there is a box with a single value,
    eliminate this value from the set of values of all its peers.

    Args:
        values: Sudoku in dictionary form.
    Returns:
        Resulting Sudoku in dictionary form after eliminating values.
    """
    solved_values = [box for box in values.keys() if len(values[box]) == 1]
    for box in solved_values:
        digit = values[box]
        for peer in peers[box]:
            values[peer] = values[peer].replace(digit,'')
    return values

```
- only choice:在图二中A4到B6所构成的3*3units中，我们发现A6中1在其他所有待填box中都没有出现，因此我们可以断定A6的值为1。这种从待填box已有取值情况，判断出唯一解的思路，就是only choice。
![](http://i.imgur.com/yHWEDzt.jpg)
python代码实现only choice：

```
from utils import *

def only_choice(values):
    """Finalize all values that are the only choice for a unit.

    Go through all the units, and whenever there is a unit with a value
    that only fits in one box, assign the value to this box.

    Input: Sudoku in dictionary form.
    Output: Resulting Sudoku in dictionary form after filling in only choices.
    """
    for unit in unitlist:
        for digit in '123456789':
            dplaces = [box for box in unit if digit in values[box]]
            if len(dplaces) == 1:
                values[dplaces[0]] = digit
    return values
```

现在将eliminate与only choice一起使用，实现Constraint Propagation，python代码如下：

```
from utils import *

def reduce_puzzle(values):
    """
    Iterate eliminate() and only_choice(). If at some point, there is a box with no available values, return False.
    If the sudoku is solved, return the sudoku.
    If after an iteration of both functions, the sudoku remains the same, return the sudoku.
    Input: A sudoku in dictionary form.
    Output: The resulting sudoku in dictionary form.
    """
    stalled = False
    while not stalled:
        # Check how many boxes have a determined value
        solved_values_before = len([box for box in values.keys() if len(values[box]) == 1])
        # Use the Eliminate Strategy
        values = eliminate(values)
        # Use the Only Choice Strategy
        values = only_choice(values)
        # Check how many boxes have a determined value, to compare
        solved_values_after = len([box for box in values.keys() if len(values[box]) == 1])
        # If no new values were added, stop the loop.
        stalled = solved_values_before == solved_values_after
        # Sanity check, return False if there is a box with zero available values:
        if len([box for box in values.keys() if len(values[box]) == 0]):
            return False
    return values

```

#### 3.2 search
对于复杂的数独问题，在Constraint Propagation之后，仍然还有box未得到答案。这里我们就需要用到搜索（search）的技巧。如图三，我们看到仍有相当多的格子仍待解决，如何选取box？答案是选possible value最少的那个，即G2。我们将8和9分别填入G2中，然后对两个新的数独使用Constraint Propagation，若还存在未解决box，继续使用search，知道所有box都填入答案。这是一个深度优先搜索，过程如图四所示。
图三
![](http://i.imgur.com/xgGTTJE.jpg)
图四
![](http://i.imgur.com/JpvYCTS.jpg)
python实现search代码如下：

```
from utils import *

def search(values):
    "Using depth-first search and propagation, try all possible values."
    # First, reduce the puzzle using the previous function
    values = reduce_puzzle(values)
    if values is False:
        return False ## Failed earlier
    if all(len(values[s]) == 1 for s in boxes): 
        return values ## Solved!
    # Choose one of the unfilled squares with the fewest possibilities
    n,s = min((len(values[s]), s) for s in boxes if len(values[s]) > 1)
    # Now use recurrence to solve each one of the resulting sudokus, and 
    for value in values[s]:
        new_sudoku = values.copy()
        new_sudoku[s] = value
        attempt = search(new_sudoku)
        if attempt:
            return attempt
```

***
到这里，数独这一节的课程内容结束，后续不上这一节的项目总结。