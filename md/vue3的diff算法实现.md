# Vue3 diff算法的实现

之前详细的说了首次渲染的流程和响应式的实现.

当响应式数据变化时,会触发依赖更新,也就是执行组件更新函数,其内部会走更新的逻辑,获取新的虚拟dom.调用patch函数与老的虚拟dom进行比对.这之后就是diff的过程,也是本文的重点.

贴上相关核心代码如下:

```js
const setupRenderEffect = (initialVNode, instance, container) => {
    // 创建渲染effect
    // 核心就是调用render，数据变化 就重新调用render 
    const componentUpdateFn = () => {
        let { proxy } = instance; //  render中的参数
        if (!instance.isMounted) {
            // 组件初始化的流程
            // 调用render方法 （渲染页面的时候会进行取值操作，那么取值的时候会进行依赖收集 ， 收集对应的effect，稍后属性变化了会重新执行当前方法）
            const subTree = instance.subTree = instance.render.call(proxy, proxy); // 渲染的时候会调用h方法
            // 真正渲染组件 其实渲染的应该是subTree
            patch(null, subTree, container); // 稍后渲染完subTree 会生成真实节点之后挂载到subTree
            initialVNode.el = subTree.el
            instance.isMounted = true;
        } else {
            // 组件更新的流程 。。。
            // 我可以做 diff算法   比较前后的两颗树 
            const prevTree = instance.subTree;
            const nextTree = instance.render.call(proxy, proxy);
            patch(prevTree, nextTree, container); // 比较两棵树
        }
    }
    const effect = new ReactiveEffect(componentUpdateFn);
    // 默认调用update方法 就会执行componentUpdateFn
    const update = effect.run.bind(effect);
    update();
}
```

+ 调用更新流程的 `patch` 后, 进入内部, 此时存在旧新节点(n1, n2),对节点进行判断(会用到位运算来判断,之后再说). 若 `n1, n2` 都不为空, 且不相等 就会进入 `processElement => patchElement` 的内部.
+ `patchElement` 主要做两件事: **更新节点自己的属性和更新子元素**

```js
const patchElement = (n1, n2) => { // old, new
    let el = n2.el = n1.el; // 先比较元素 元素一致 则复用 
    const oldProps = n1.props || {}; // 复用后比较属性
    const newProps = n2.props || {};
    // 先比对更新属性
    patchProps(oldProps, newProps, el);
    // 实现比较儿子  diff算法   我们的diff算法是同级别比较的
    patchChildren(n1, n2, el); // 用新的儿子n2 和 老的儿子n1 来进行比对  比对后更新容器元素
}
```

更新自身属性的逻辑比较好理解: 
1. 新老节点都有的情况下, 去新节点属性. (遍历新节点属性, 找对应的老节点属性, 传入hostPatchProp去做更新.)
2. 老节点有的属性, 新节点没有的, 则移除老节点属性. (遍历老节点属性, 如果当前属性key不在新节点属性之中, 则调用hostPatchProp做去除该属性操作)

```js
const patchProps = (oldProps, newProps, el) => {
    if (oldProps === newProps) return;
    for (let key in newProps) {
        const prev = oldProps[key];
        const next = newProps[key]; // 获取新老属性
        // 新老都有的情况下，取新值
        if (prev !== next) {
            hostPatchProp(el, key, prev, next);
        }
    }
    for (const key in oldProps) { // 老的有新的没有  移除老的
        if (!(key in newProps)) {
            hostPatchProp(el, key, oldProps[key], null);
        }
    }
}
```

之后就是的patchChildren逻辑, 主要可以归纳为以下几种情况的比对更新: 

1. 如果新的子元素是空， 老的子元素不为空，直接卸载 unmount 即可。
2. 如果新的子元素不为空，老的子元素是空，直接创建加载即可。
3. 如果新的子元素是文本，老的子元素如果是数组就需要全部 unmount，是文本的话就需要执行 hostSetElementText。
4. 如果新的子元素是数组，比如是使用 v-for 渲染出来的列表，老的子元素如果是空或者文本，直接 unmout 后，渲染新的数组即可。
5. 新老子元素都是数组的情况, 这也是最复杂的情况.

针对以上情况, 贴下对应的核心代码:
```js
function patchChildren(n1, n2, container) {
  const prevFlag = n1.shapeFlag
  const c1 = n1.children
  const nextFlag = n2.shapeFlag
  const c2 = n2.children
  // 新的vdom是文本, 老的vdom可以是数组, 文本, 空.
  if (nextFlag & ShapeFlags.TEXT_CHILDREN) {
    if(prevFlag & ShapeFlags.ARRAY_CHILDREN){
      // 老的vdom是数组。unmount
      c1.forEach(child=>unmount(child))
    }
    if (c2 !== c1) {
      hostSetElementText(container, c2)
    }
  } else {
    // 老的vdom是数组
    if (prevFlag & ShapeFlags.ARRAY_CHILDREN) {
      // 新的vdom也是数组，
      if (nextFlag & ShapeFlags.ARRAY_CHILDREN) {
        // 最简单粗暴的方法就是unmountChildren(c1), 再mountChildren(c2)
        // 这样所有dom都没法复用了
        // 这里也有两种情况，没写key和写了key, key就是虚拟dom的唯一标识
        // 在新老数组中的虚拟dom的key相同，就默认可以复用dom
        // <li :key="xx"></li>
        if(c1[0].key && c2[0].key){
          patchKeyedChildren(c1, c2, container, anchor, parentComponent)
        }else{
          // 么有key，只能暴力复用一个类型的dom
          patchUnKeyedChildren(c1, c2, container, anchor, parentComponent)
        }
      }else{
        // next是null
        unmountChildren(c1)
      }
    }else{
      if (nextFlag & ShapeFlags.ARRAY_CHILDREN) {
        mountChildren(c2, container)
      }
    }
  }
}
```

+ 对于第五种情况: 最朴实无华的思路就是把老的子元素全部 unmount，新的子元素全部 mount，这样虽然可以实现功能，但是没法复用已经存在的 DOM 元素，比如我们只是在数组中间新增了一个数据，全部 DOM 都销毁就有点太可惜了。

+ 所以，**我们需要判断出可以复用的 DOM 元素，如果一个虚拟 DOM 没有改动或者属性变了，不需要完全销毁重建，而是更新一下属性，最大化减少 DOM 的操作**. 这也是diff的目的

接下来, 重点分析patchKeyedChildren的实现, 这也是最复杂的情况5: 新老节点的子元素都是数组的情况. 它的目的很简单: **尽可能高效地把老的子元素更新成新的子元素**


## patchKeyedChildren的实现

首先, 比对c1, c2.的首尾部分. 设置指针: i, e1, e2. 其中 i 指向c1, c2的首部, e1, e2指向 c1, c2的尾部. 

1. (sync from start and end) 从首部开始遍历一一比对, 如果两个子节点相同则可以复用(调用patch继续下一个层级的比对), i++ 继续下一对子节点的比较. 如果两个子节点不一致, 则不可复用, 跳出循环. 开始尾部比较, 逻辑与此基本类似, 只不过 `i++` 变为 `e1--`, `e2--`
2. (common sequence + mount) 经过步骤1, 跳出循环之后. 如果出现` i > e1` 的情况, 则说明是新增, 并且 i 到 e2 之间都是新增的节点(闭区间). 所以思路就是将该区间的节点依次挂载到container之上即可. 但是要注意, 新增插入的位置需要商榷: 需要找到一个参考节点, 将新增的节点插入到这个参考节点之前. 至于这个参考节点如何确定?? 
    + 参考节点: 通过e2的下一个节点是否小于c2的长度来判断, 若小于, 则参考点就为`e2.nextNode`. 若大于等于, 则参考点为null. 可以理解为直接在container尾部追加即可.
3. (common sequence + unmount) 经过步骤1, 跳出循环之后. 如果出现` i > e2` 的情况, 则说明是删除节点, 并且 i 到 e1 之间都是要删除的节点(闭区间). 所以思路就是将该区间的节点依次unmount即可. 删除不需要找参考点.
4. 接下来就是最复杂的乱序比对(unknown sequence). 假设经过步骤1之后, 中间有一小段区间新旧节点有增加,删除,位置移动的情况, 就符合该步骤要解决的问题.
    + 先将该区间内的新节点序列的key和index组成一个map映射. 再创建一个数组N, 记录区间内新节点是需要新增还是只需要移动复用. 初始化该数组N长度为该区间长度, 初始值都设为0, 0则代表对应的节点是新增节点.
    + 再遍历中间区间内老节点, 若当前老节点可以在刚刚创建的新节点映射中找到对应的key, 则说明可以复用(调用patch继续下一个层级的比对), 同时, 新节点数组N对应的位置设置为老节点的索引. 若当前老节点找不到对应的key, 则说明该老节点需要删除. 
    + 以上步骤完成之后, 该数组N其中有的为0, 有的不为0. 0 则代表对应的新节点是新增节点, 不为0 则代表: 可以复用的节点, 可能只是位置的改变,或者位置可能也不变. 求得该数组N的**最长递增子序列**的索引. 目的是最长递增子序列对应的节点的位置都不用改变，直接复用即可。
    + 倒序遍历数组N，为0的情况，则直接新增节点，并插入container中。不为0的，跟子序列对应索引比对，一致则不需要任何操作，不一致，则只需要做移动操作即可，不用新建节点。

以上就是vue3的diff过程，有些许复杂，贴上核心的代码实现，以助于理解。

```js
 const patchKeyedChildren = (c1, c2, container) => {
    let e1 = c1.length - 1;
    let e2 = c2.length - 1;
    let i = 0; // 从头开始比较
    // 1.sync from start 从头开始一个个孩子来比较 , 遇到不同的节点就停止了
    while (i <= e1 && i <= e2) { // 如果i 和 新的列表或者老的列表指针重合说明就比较完毕了
        const n1 = c1[i];
        const n2 = c2[i];
        if (isSameVNodeType(n1, n2)) { // 如果两个节点是相同节点 则需要递归比较孩子和自身的属性
            patch(n1, n2, container)
        } else {
            break;
        }
        i++;
    }
    // sync from end
    while (i <= e1 && i <= e2) { // 如果i 和 新的列表或者老的列表指针重合说明就比较完毕了
        const n1 = c1[e1];
        const n2 = c2[e2];
        if (isSameVNodeType(n1, n2)) { // 如果两个节点是相同节点 则需要递归比较孩子和自身的属性
            patch(n1, n2, container)
        } else {
            break;
        }
        e1--;
        e2--
    }
    console.log(i, e1, e2); // 确定好了 头部 和 尾部相同的节点 定位到除了头部和尾部的节点
    // 3.common sequence + mount 新增情况
    if (i > e1) { // 看i和e1 之间的关系 如果i 大于 e1  说明有新增的元素
        if (i <= e2) {  // i和 e2 之间的内容就是新增的
            const nextPos = e2 + 1;
            // 取e2 的下一个元素 如果下一个没有 则长度和当前c2长度相同  说明追加
            // 取e2 的下一个元素 如果下一个有 说明要在头部追加 则取出下一个节点作为参照物
            const anchor = nextPos < c2.length ? c2[nextPos].el : null;
            // 参照物的目的 要计算是向前插入还是向后插入
            while (i <= e2) {
                patch(null, c2[i], container, anchor); // 没有参照物 就是appendChild
                i++;
            }
        }
        // 4.common sequence + unmount 删除情况
    } else if (i > e2) {   // 看一下 i 和 e2 的关系 如果 e2 比i小 说明 老的多新的少
        while (i <= e1) {
            // i 和 e1 之间的就是要删除的
            unmount(c1[i]);
            i++;
        }
    }
    // unknown sequence
    const s1 = i;  // s1 -> e1 老的孩子列表
    const s2 = i;  // s2 -> e2  新的孩子列表
    // 根据新的节点 创造一个映射表 ， 用老的列表去里面找有没有，如果有则复用，没有就删除。 最后新的多余在追加
    const keyToNewIndexMap = new Map(); // 这个目的是为了可以用老的来查看有没有新的
    for (let i = s2; i <= e2; i++) {
        const child = c2[i];
        keyToNewIndexMap.set(child.key, i)
    }
    const toBepatched = e2 - s2 + 1; // 4
    const newIndexToOldMapIndex = new Array(toBepatched).fill(0); // 最长递增子序列会用到这个列表  5 3 4 0
    // 拿老的去新的中查找
    // 找到一样的需要patch
    for (let i = s1; i <= e1; i++) { // 新的索引映射到老的索引的映射表
        const prevChild = c1[i]; // 拿到老的每一个节点
        let newIndex = keyToNewIndexMap.get(prevChild.key);
        if (newIndex == undefined) { // 删掉老的多余的
            unmount(prevChild)
        } else {
            newIndexToOldMapIndex[newIndex - s2] = i + 1;// 保证填的肯定不是0 , 0意味着添加了一个元素
            // 比较两个人的节点 
            patch(prevChild, c2[newIndex], container); // 填表后 还要比对属性和儿子
        }
    }
    // 在去移动需要移动的元素
    let queue = getSequence(newIndexToOldMapIndex); // 求出队列   [1,2]  1 ,2 不用动
    let j = queue.length - 1; // 拿到最长递增子序列的末尾索引
    for (let i = toBepatched - 1; i >= 0; i--) {
        let lastIndex = s2 + i; // h的索引
        let lastChild = c2[lastIndex];
        let anchor = lastIndex + 1 < c2.length ? c2[lastIndex + 1].el : null // 参考点
        if (newIndexToOldMapIndex[i] == 0) { // 等于0的时候还没有真实节点，需要创建真实节点在插入
            patch(null, lastChild, container, anchor); // 创建一个h 插入到 f的前面
        } else {
            // 这里可以进行优化 问题出在可能有一些节点不需要移动，但是还是全部插入了一遍
            // 性能消耗， 最长递增子序列 减少dom的插入操作 
            if (i !== queue[j]) {
                // 3 2 1 0  倒叙插入 所以  i的值 就是  3 2 1 0
                hostInsert(lastChild.el, container, anchor); // 将列表倒序的插入
            }else{
                j--; // 这里做了一个优化 表示元素不需要移动了
            }
        }
    }
}
```