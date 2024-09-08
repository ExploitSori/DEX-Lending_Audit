# Upside Academy Lending Audit

## Audit repositories  
24.09.08. 이전 commit.  
https://github.com/hakid29/Lending_solidity - hakid  
https://github.com/Entropy1110/Lending_solidity - entropy  
https://github.com/dokpark21/DEX_Lending_solidity - kyrie  
https://github.com/pluto1011/-Lending-DEX-_solidity - icarus  
https://github.com/rivercastleone/Lending_solidity - castle  
https://github.com/je1att0/Lending_solidity - jacqueline  
https://github.com/Null0RM/Lending_solidity - nullorm  
https://github.com/55hnnn/Lending_solidity - kenny  
https://github.com/0xGh-st/Lending_solidity - ghost  
https://github.com/GODMuang/Lending_solidity - muang  

## Icarus
### Case 1
### 파급력  
| <span style="color:Orange"> Medium</span>  
### 설명  
```javascript
function getAccruedSupplyAmount(address user) external view returns (uint256) {
    //수수료가 복리인 거 같은데 그냥 if문으로,,,,,
    uint256 acc = userAccounts[msg.sender].usdcCollateral;
    if (block.number == 7200001) {
        if (acc == 30000000 ether) {
            return 30000792 ether; //1000일 후 user는 이 친구밖에 없음
        }
    }

    if (block.number == 7200001 + 3600000) {
        if (acc == 30000000 ether) {
            if (userList.length == 2) {
                return 30001605 * 1e18;
            } else {
                return 30001547 * 1e18;
            } //1000일 후 user는 이 친구밖에 없음
        }
        if (acc == 100000000 ether) {
            return 100005158 * 1e18;
        }
        if (acc == 10000000 ether) {
            return 10000251 * 1e18;
        }
    }
}
```
block number에 따라 이자율이 정해지는 방식, 현재 상태에서는 특정 블록에서만 이자가 발생하게 되고, 모든 분기가 초기 if에 결정되어 해당 함수를 호출했을 때 조건에 해당하는 블록이 아니라면 함수가 동작하지 않는다. 또한 block이 생성되는 주기가 바뀌거나 할 때 side effect가 발생할 수 있다.  
의도를 정확하게 확립하여 이자율 계산이 명확한 조건에 이루어질 수 있도록 해야한다.

### Case2
### 파급력  
| <span style="color:yellowgreen"> informational</span> 
### 설명  
```javascript  
function initializeLendingProtocol(address token) external payable {
        require(totalEthSupply == 0 && totalUsdcSupply == 0, "Already initialized");
        require(token == usdcToken, "Unsupported token");
        require(msg.value == 1, "Incorrect ETH amount");

        totalEthSupply = 1;
    }
```
해당 함수가 external로 설정되어 있고 여러번 호출 가능하므로 totalEthSupply가 1로 설정되어 deposit, withdraw 등 해당 변수를 사용하는 함수에서 문제가 발생할 수 있다. 1번만 호출할 수 있도록 설정이 필요할 것이다.

## Nullorm
### Case 1
### 파급력
| <span style="color:yellowgreen"> informational</span>
### 설명  
```javascript
function _depositUpdate() internal {
    uint length = users.length;
    uint timeElapsed;
    uint calc;
    uint i;
    User memory user;

    for(i = 0; i < length; i++) {
        user = user_info[users[i]];
        
        timeElapsed = (block.number - user.deposit_update_time) / 7200;
        if (timeElapsed == 0)
            continue;

        calc = _pow(total_borrowed, block.number / 7200 , true) - _pow(total_borrowed, user.deposit_update_time / 7200, true);
        calc = calc * user.deposit_usdc / total_deposited_USDC;
        user.last_accrued_interest += calc;

        user.deposit_update_time = block.number;
        user_info[users[i]] = user;
    }
}
```
반복문을 실행할 때 length가 모든 user를 조회하는데, 사용자 수가 많아지면 가스비가 과도하게 소모될 수 있다.
### Kyrie
### Case 1
### 파급력
| <span style="color:Red"> High</span>  
### 설명
```javascript
function liquidate(
    address user,
    address tokenAddress,
    uint256 amount
) external {
    uint256 ethPrice = oracle.getPrice(address(0x0));
    uint256 usdcPrice = oracle.getPrice(address(usdc));

    _borrowUSDCs[user].amount = _calculateInterest(
        _borrowUSDCs[user].amount,
        _borrowUSDCs[user].blockNumber,
        INTEREST_RATE
    );

    _borrowUSDCs[user].blockNumber = block.number;

    uint256 borrowValue = (_borrowUSDCs[user].amount * usdcPrice) /
        ethPrice;

    uint256 remainingCollateralValue = ((_depositETHs[user]) * LT) / 100;
    require(
        remainingCollateralValue < borrowValue,
        "Sufficient collateral"
    );
    require(
        amount == (_borrowUSDCs[user].amount * 1) / 4,
        "Invalid amount"
    );

    require(usdc.balanceOf(address(this)) >= amount, "Insufficient funds");

    _depositETHs[user] -= (amount * usdcPrice) / ethPrice;
    _borrowUSDCs[user].amount -= amount;
    totalBorrowedUSDCs -= amount;
}
```
tokenAddress를 사용하지 않고 contract에서 정의한 usdc.balanceOf(address(this)를 이용해 지갑에 있는 금액을 확인 한뒤 컨트랙트 내에서만 청산하고 실제 토큰을 청산하지 않는다.

### Muang
### Case 1
### 파급력
| <span style="color:Red"> High</span>  
### 설명
```javascript
function repay(address _token, uint256 _amount) external { 
    updateDebt(msg.sender,token);
    require(_amount <= userBorrowed[msg.sender][_token], "you dont have to pay this much hehe..");
    require(IERC20(_token).balanceOf(msg.sender) >= _amount, "INSUFFICIENT_TOKEN_TO_REPAY");

    IERC20(_token).transferFrom(msg.sender, address(this), _amount);
    userBorrowed[msg.sender][_token] -= _amount;
    totalDepositToken[_token] +=_amount;
}
```
repay 함수 호출 시 _token에 존재하는 양 확인 및 회수하기 때문에 빌린 토큰과 관계없이 상환이 가능하다. token address를 확인해야 한다.  

### Kaymin
### Case 1
### 파급력
| <span style="color:yellowgreen"> informational</span>  
### 설명
```javascript
function liquidate(address user, address token, uint256 amount) external {
    require(token == address(usdc), "Only USDC !!");

    _updateInterest(user); 
    uint max_liquidation;
    if (users[user].borrowed_usdc>=100 ether){
        max_liquidation=users[user].borrowed_usdc/4;
    }
    else{
        max_liquidation=users[user].borrowed_usdc;
    }
    ...
}
```
users[user].borrowed_usdc>=100 ether 인경우에 25%로 계산하게 되어있는, 이벤트 등의 로직이 필요하지 않고 테스트케이스 상 100 ether 미만인 경우가 존재하지 않아. 해당 분기는 필요 없는 분기이다.

### jacqueline
### Case 1
### 파급력
| <span style="color:Brown"> Critical</span>  
### 설명  
```javascript
function withdraw (address _tokenAddress, uint256 _amount) public payable {
    ...
    if (_tokenAddress == address(0x0)) {
    ...
    } else {
        usdc.transfer(msg.sender, _amount);
    }
    ...
}
```
external로 호출할 시 token의 address가 0이 아닐 때 amount 만큼 transfer 받을 수 있다. deposit 한 자원을 회수하려고 하는 로직을 추가로 구현해 주어야 한다.