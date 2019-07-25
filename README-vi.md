
![](https://images.viblo.asia/9bc39566-d38e-4002-8965-1ff9a9d8cc53.jpg)


### Lời gọi bên ngoài (External Calls)


#### Hãy thật cẩn trọng khi sử dụng external calls

Các message gọi đến những hợp đồng không đáng tin cậy có thể gây ra một số rủi ro hoặc lỗi không mong muốn. Các lời gọi bên ngoài có thể thực thi mã độc trong hợp đồng đó hoặc bất kỳ hợp đồng nào khác mà nó phụ thuộc vào. Như vậy, mọi lời gọi bên ngoài nên được xem là ẩn chứa rủi ro bảo mật. Trong trường hợp bất khả kháng, hãy sử dụng các đề xuất dưới đây để giảm thiểu rủi ro có thể xảy ra.

#### Đánh dấu các hợp đồng không đáng tin cậy

Khi tương tác với các lời gọi bên ngoài, tên các biến, phương thức và các interface nên được đặt sao cho nó thể hiện được việc tương tác với các lời gọi từ bên ngoài có an toàn hay là không ? Điều này áp dụng cho các hàm mà nó có thể được gọi từ các hợp đồng bên ngoài.

```javascript
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted

function makeWithdrawal(uint amount) { // Isn't clear that this function is potentially unsafe
    Bank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // untrusted external call
TrustedBank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

#### Tránh thay đổi trạng thái sau các lời gọi từ bên ngoài

Khi sử dụng các lời gọi mặc địch (raw calls) như **someAddress.call()** hoặc contract calls (**ExternalContract.someMethod()**) thì mã độc có thể được thi. Thậm chí nếu ExternalContract không có mã độc, mã độc có thể được thực thi bởi bất cứ hợp đồng thông minh nào mà ExternalContract gọi.

Mã độc khi thực thi có thể chiếm quyền kiểm soát hợp đồng, tiêu biểu là lỗ hổng Reentrancy.

Nếu bạn đang thực hiện lời gọi đến một hợp đồng bên ngoài không đáng tin cậy, hãy tránh các thay đổi trạng thái sau lời gọi. Nguyên tắc này đôi khi được gọi với các tên checks-effects-interactions pattern.

#### Sự khác nhau giữa send (), transfer () và call.value ()

Khi thực hiện một giao dịch từ hợp đồng thông minh, cần phân biệt sự giống và khác giữa **someAddress.send()**, **someAddress.transfer()**, **someAddress.call().value()**.

- someAddress.send() và someAddress.transfer() được coi là an toàn để chống lại reentrancy. Chúng giới hạn 2.300 gas, chỉ đủ để ghi lại một sự kiện thay vì chạy một đoạn mã khai thác.

- x.transfer(y) tương đương với lệnh x.send (y), nó sẽ tự động revert nếu giao dịch thất bại.

- Khác với someAddress.send() và someAddress.transfer(), someAddress.call.value(y) không giới hạn gas cho lời gọi và do đó hacker có thể thực thi lời gọi đến một đoạn mã độc nhằm mục đích xấu. Do đó, nó không an toàn để chống lại reentrancy.

Sử dụng send() hoặc transfer() sẽ ngăn chặn reentrancy nhưng nó sẽ không thích hợp với các hợp đồng mà fallback function yêu cầu hơn 2.300 gas. Chúng ta cũng có thể sử dụng someAddress.call.value(ethAmount) .gas(gasAmount) để giới hạn lượng gas cho lời gọi một cách tùy ý.

#### Xử lý lỗi từ các lời gọi bên ngoài

Solidity cung cấp các phương thức gọi mức thấp (low level) : address.call(), address.callcode(), address.delegatecall() và address.send(). Các phương thức ở mức thấp này không bao giờ ném ra ngoại lệ (throw an exception), nhưng sẽ trả về false nếu lời gọi gặp phải ngoại lệ. Mặt khác, các lời gọi hợp đồng (contract calls) (ví dụ như ExternalContract.doSomething()) sẽ tự động ném ra một ngoại lệ và báo lỗi.

Nếu bạn lựa chọn sử dụng các phương thức gọi ở mức thấp, hãy kiểm tra xem lời gọi sẽ thất bại hay thành công, bằng cách kiểm tra giá trị trả về là true hay false.

```javascript
// bad
someAddress.send(55);
someAddress.call.value(55)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3('deposit()'))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted

// good
if (!someAddress.send(55)) {
  // handle failure code
}

ExternalContract(someAddress).deposit.value(100);
```

#### Ưu tiên pull hơn là push cho các external call

Các lời gọi từ bên ngoài có thể thất bại một cách vô tình hoặc cố ý. Để giảm thiểu rủi ro từ các lỗi đó gây ra, tốt hơn hết là chia từng lời gọi thành các lời gọi nhỏ hơn. Điều này đặc biệt phù hợp với các giao dịch thanh toán, trong đó cho phép người dùng rút tiền sẽ tốt hơn là tự động chuyển tiền cho họ. (Điều này cũng làm giảm khả năng xảy ra sự cố với gasLimit.) và tránh việc thực hiện cùng một lúc nhiều hàm transfer() trong một giao dịch.

```javascript
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() payable {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            highestBidder.transfer(highestBid); // if this call consistently fails, no one else can bid
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            refunds[highestBidder] += highestBid; // record the refund that this user can claim
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        msg.sender.transfer(refund);
    }
}
```

#### Không nên dùng delegatecall với đoạn mã không được tin cậy

Hàm delegatecall được sử dụng để gọi các hàm từ các hợp đồng khác như thể chúng thuộc về hợp đồng của người gọi. Do đó, người gọi có thể thay đổi trạng thái của hợp đồng được gọi đến. Điều này có thể không an toàn. Ví dụ dưới đây cho thấy cách sử dụng delegatecall có thể dẫn đến việc hợp đồng bị phá hủy và mất hết số dư.

```javascript
contract Destructor
{
    function doWork() external
    {
        selfdestruct(0);
    }
}

contract Worker
{
    function doWork(address _internalWorker) public
    {
        // unsafe
        _internalWorker.delegatecall(bytes4(keccak256("doWork()")));
    }
}
```

Nếu **worker.doWork()** được gọi với tham số là địa chỉ của hợp đồng Destructor, hợp đồng Worker sẽ tự hủy. Bạn chỉ nên thực hiện delegate call cho các hợp đồng đáng tin cậy.

**Lưu ý**: Đừng cho rằng các hợp đồng khi được khởi tạo có số dư bằng 0. Một kẻ tấn công có thể gửi ether đến địa chỉ của hợp đồng trước khi nó được khởi tạo. [Xem vấn đề 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61) để biết thêm chi tiết.

### Ether có thể được gửi đến bất kỳ hợp đồng nào

Kẻ tấn công có thể gửi ether đến bất kỳ tài khoản nào và điều này không thể ngăn chặn được (ngay cả với fallback function với câu lệnh revert).

Kẻ tấn công có thể làm điều này bằng cách tạo ra một hợp đồng, gửi cho nó 1 wei và hàm selfdestruct(victimAddress), ở đây victimAddress là địa chỉ hợp đồng cần gửi ether vào. 

### Hãy nhớ rằng Ethereum là mạng public blockchain, mọi dữ liệu trên các block đều được công khai

Nhiều ứng dụng yêu cầu dữ liệu được gửi phải ở chế độ riêng tư cho đến một lúc nào đó. Các trò chơi (ví dụ: oản tù tì) và việc đấu giá kín là hai ví dụ chính. Nếu bạn đang xây dựng một ứng dụng mà sự riêng tư là một vấn đề, hãy đảm bảo bạn tránh yêu cầu người dùng công khai thông tin quá sớm. Chiến lược tốt nhất là chia thành các giai đoạn riêng biệt: đầu tiên thì sử dụng hàm băm của các giá trị và trong giai đoạn tiếp theo thì tiết lộ các giá trị.

Ví dụ:

- Trong trò chơi oản tù tì, yêu cầu cả hai người chơi gửi giá trị băm của kéo, đá hay giấy (do người chơi quyết định), sau đó trò chơi yêu cầu cả hai người chơi gửi kết quả mình lựa chọn. Tiếp đó so sánh giá trị băm, nếu khớp thì hợp lệ, trò chơi sẽ phân thắng hòa hay thua dựa trên kết quả chọn của 2 người chơi.

- Trong phiên đấu giá kín, yêu cầu người đấu giá gửi giá trị băm mức giá mà họ chọn trong giai đoạn ban đầu (cùng với khoản tiền gửi lớn hơn giá trị giá thầu của họ), sau đó gửi giá trị đấu giá của họ trong giai đoạn thứ hai.

- Khi phát triển một ứng dụng mang tính ngẫu nhiên, thứ tự phải luôn là: (1) người chơi submit, (2) số ngẫu nhiên được tạo, (3) người chơi hoàn thành giao dịch. Phương thức mà các số ngẫu nhiên được tạo ra là cả một lĩnh vực nghiên cứu, các giải pháp tốt nhất hiện tại bao gồm việc sử dụng block header Bitcoin (được xác minh thông qua http://btcrelay.org), các cơ chế hash-commit-reveal (tức là một bên tạo ra một số, xuất bản hàm băm của nó để "cam kết" và sau đó tiết lộ giá trị sau), cùng với đó là [RANDAO](https://github.com/randao/randao). Vì Ethereum là một giao thức xác định, không có biến nào trong giao thức có thể được sử dụng như một số ngẫu nhiên không thể đoán trước. Ngoài ra, hãy lưu ý rằng thợ đào (miner) trong một chừng mực nào đó kiểm soát giá trị block.blockhash().

### Cảnh giác với khả năng một số người tham gia có thể "drop offline" và không quay lại

Ví dụ, trong trò chơi oản tù tì, một ván đấu được tiếp tục cho đến khi cả hai người chơi gửi lựa chọn của họ. Tuy nhiên, một người chơi có thể không bao giờ gửi lựa chọn của họ - thực tế, nếu một người chơi thấy động thái được tiết lộ từ người chơi khác và xác định rằng họ đã thua, họ không có lý do gì để tự gửi kết quả. Khi gặp các tình huống như vậy thì , (1) cung cấp một cách để tránh những người chơi không tham gia, có thể giới hạn thời gian và (2) xem xét thêm lợi ích bổ sung cho những người tham gia khi gửi kết quả trong tất cả các tình huống.

### Trường hợp đổi dấu số âm bé nhất

Solidity cung cấp một số kiểu dữ liệu số nguyên. Giống như trong hầu hết các ngôn ngữ lập trình khác, trong Solidity, một số nguyên N bit có thể biểu thị các giá trị từ -2 ^ (N-1) đến 2 ^ (N-1) - 1. Điều này có nghĩa là không có giá trị dương mà có trị tuyệt đối bằng MIN_INT. - MIN_INT sẽ được gắn bằng MIN_INT.

Điều này đúng với tất cả các kiểu số nguyên trong Solidity (int8, int16, ..., int256).

```javascript
contract Negation {
    function negate8(int8 _i) public pure returns(int8) {
        return -_i;
    }

    function negate16(int16 _i) public pure returns(int16) {
        return -_i;
    }

    int8 public a = negate8(-128); // -128
    int16 public b = negate16(-128); // 128
    int16 public c = negate16(-32768); // -32768
}
}

```

Một cách để xử lý điều này là kiểm tra giá trị của biến trước khi đảo dấu và ném ra ngoại lệ nếu nó bằng MIN_INT. Một tùy chọn khác là đảm bảo rằng số âm nhất bé nhất sẽ không bao giờ đạt được bằng cách sử kiểu biến có khoảng giá trị lớn (ví dụ: int32 thay vì int16).

### Sử dụng assert(), revert() và require() đúng cách

Các hàm assert và require được sử dụng để kiểm tra các điều kiện và ném ra một ngoại lệ nếu điều kiện không được đáp ứng.

Hàm **assert** chỉ nên được sử dụng để kiểm tra các lỗi bên cục bộ và để kiểm tra các biến hằng.

Hàm **require** nên được dùng để đảm bảo các điều kiện hợp lệ, chẳng hạn như biến đầu vào, biến trạng thái của hợp đồng hoặc để xác thực giá trị trả về từ các lời gọi đến hợp đồng bên ngoài.

Ví dụ dưới đây cho thấy rằng các opcode không hợp lệ không có cơ hội để thực thi: các biến đều được xác minh và nếu có sai số thì đoạn mã sẽ ném ra lỗi.

```javascript
pragma solidity ^0.5.0;

contract Sharer {
    function sendHalf(address payable addr) public payable returns (uint balance) {
        require(msg.value % 2 == 0, "Even value required."); //Require() can have an optional message string
        uint balanceBeforeTransfer = address(this).balance;
        addr.transfer(msg.value / 2);
        // Since transfer throws an exception on failure and
        // cannot call back here, there should be no way for us to
        // still have half of the money.
        assert(address(this).balance == balanceBeforeTransfer - msg.value / 2); // used for internal error checking
        return address(this).balance;
    }
}
```

### Chỉ sử modifier khi cần kiểm tra dữ liệu

Mã bên trong modifier được thực thi trước khi chạy mã bên trong hàm. Do đó, bất kỳ thay đổi trạng thái hoặc lời gọi bên ngoài nào được tạo ra bởi đoạn mã trong modifier cũng sẽ vi phạm thiết kế [Checks-Effects-Interactions](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) mà chúng tôi đã đề ra. Ví dụ dưới đây, một lời gọi bên ngoài hợp đồng được chèn trong modifier có thể dẫn đến lỗ hổng reentrancy.

```javascript
contract Registry {
    address owner;

    function isVoter(address _addr) external returns(bool) {
        // Code
    }
}

contract Election {
    Registry registry;

    modifier isEligible(address _addr) {
        require(registry.isVoter(_addr));
        _;
    }

    function vote() isEligible(msg.sender) public {
        // Code
    }
}
```

Trong trường hợp này, hợp đồng Registry có thể bị tấn công reentracy bằng cách gọi đến Election.vote()

**Lưu ý**: Sử dụng modifier để thay các câu lệnh kiểm tra điều kiện bên trong thân của hàm. Điều này làm cho mã nguồn hợp đồng thông minh của bạn gọn nhẹ và dễ đọc hơn.

### Hãy cẩn thận với việc làm tròn kết quả trong phép chia

Tất cả các phép chia số nguyên được làm tròn bằng cách lấy số nguyên gần nhất. Nếu bạn cần độ chính xác cao hơn, hãy cân nhắc lưu trữ cả tử và mẫu số, hoặc số nhân.

(Trong tương lai, Solidity sẽ có [fixed_point type](https://solidity.readthedocs.io/en/develop/types.html#fixed-point-numbers), điều này sẽ khiến vấn đề dễ dàng hơn.)

```javascript
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer
```

Sử dụng số nhân sẽ ngăn việc làm tròn xuống, số nhân này cần được tính toán khi làm việc với x trong tương lai:

```javascript
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

Lưu trữ tử số và mẫu số có nghĩa là bạn có thể tính kết quả của tử số/mẫu số ngoài chuỗi:

```javascript
// good
uint numerator = 5;
uint denominator = 2;
```

### Abstract contract và interfaces

Cả interface và hợp đồng trừu tượng (abstract contract) đều có chung một tư tưởng là cho phép tùy chỉnh mã nguồn của các function dựa trên prototype có sẵn. Interface, được giới thiệu trong phiên bản Solidity 0.4.11, tương tự như các abstract contract nhưng interface chỉ có các prototype mà không có chứa thân hàm. Interface cũng có những hạn chế như không thể truy cập vào storage hoặc kế thừa từ các interface khác, điều này làm cho các abstract contract có ưu thế hơn một chút. Ngoài ra, điều quan trọng cần lưu ý là nếu một hợp đồng kế thừa từ một abstract contract thì các hàm sẽ được thực thi bằng cách ghi đè (overriding).

### Fallback functions

#### Giữ cho Fallback function đơn giản

[Fallback function](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) được thực thi khi hợp đồng được gọi bởi một message không có tham số (hoặc message đó gọi đến một hàm không tồn tại trong hợp đồng). Nếu bạn chỉ muốn nhận Ether từ fallback function bằng cách gọi .send() hoặc .transfer(), thì 2300 gas đủ để cho bạn kích hoạt một event. Nếu cần sử nhiều tính toán hơn thì có thể cấu hình lượng gas tối đa mà fallback function có thể sử dụng.

```javascript
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

### Kiểm tra độ dài dữ liệu trong fallback function

Fallback function không chỉ được dùng để nhận ether gửi vào hợp đồng (không có dữ liệu trong message) mà còn dùng kh gọi hàm không có trong hợp đồng hoặc tham số không đúng. Do đó, kiểm tra độ dài data trước khi thực thi các mã trong fallback function nhằm tránh việc bị thực thi mã độc.

```javascript
// bad
function() payable { emit LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

### Định nghĩa rõ ràng các hàm  và các biến có thể nhận ether

Bắt đầu từ phiên bản Solidity 0.4.0, mọi hàm nhận ether phải có modifier ```payable```, mặt khác, nếu lời gọi đến hàm payable có msg.value = 0 thì giao dịch sẽ bị revert ([trừ khi bị bắt buộc](https://consensys.github.io/smart-contract-best-practices/recommendations/#remember-that-ether-can-be-forcibly-sent-to-an-account)).

 Nếu bạn muốn dùng chức năng chuyển tiền, hãy khai báo các biến và các tham số của hàm có dạng ```address payable```. Bạn chỉ có thể sử dụng .transfer (..) và .send (..) trên ```address payable``` thay vì ```address```. Bạn có thể sử dụng .call (..) cho cả ```address payable``` và ```address```. Nhưng điều này không được khuyến khích.

**Lưu ý**: Modifier payable chỉ áp dụng cho các lời gọi từ bên ngoài. Nếu một hàm non-payable gọi hàm payable trên cùng một hợp đồng, thì hàm non-payable sẽ không thành công, mặc dù msg.value vẫn được đặt.

### Định nghĩa rõ ràng phạm vi truy cập của các hàm, các biến

Các hàm có bốn loại phạm vi truy cập là external, public, private, internal. . Đối với các biến, không thể định nghĩa phạm vi external. Định nghĩa đầy đủ, rõ ràng phạm vi truy cập của các biến, các hàm giúp dễ dàng nắm được được phạm vi của từng thành phần trong hợp đồng, tránh các lỗi không đáng có.

- Các hàm external là một phần chức năng của contract interface. Các hàm external hiểu quả hơn các hàm public khi tham số là các mảng dữ liệu lớn do hàm external sẽ tốn ít gas hơn.
- Các hàm public có thể được gọi từ bất cứ đâu, trong hợp đồng, hoặc từ một hợp đồng khác.
- Các hàm internal chỉ có thể được gọi từ bên trong hợp đồng hoặc các hợp đồng kế thừa.
- Các hàm private chỉ có thể được gọi từ bên trong hợp đồng.

```javascript
// bad
uint x; // the default is internal for state variables, but it should be made explicit
function buy() { // the default is public
    // public code
}

// good
uint private y;
function buy() external {
    // only callable externally or using this.buy()
}

function utility() public {
    // callable externally, as well as internally: changing this code requires thinking about both cases.
}

function internalAction() internal {
    // internal code
}
```

### Fix cứng phiên bản trình biên dịch của Solidity

Các hợp đồng nên được triển khai với cùng phiên bản trình biên dịch mà chúng đã được kiểm thử nhiều nhất. Fix cứng phiên bản trình biên dịch để tránh trường hợp được triển khai bởi một phiên bản mới hơn (vốn có thể tiềm ẩn lỗi).

```
// bad
pragma solidity ^0.4.4;

// good
pragma solidity 0.4.4;
```

### Sử dụng các sự kiện (event) để theo dõi hoạt động của hợp đồng

Cần có cách giám sát hoạt động của hợp đồng sau khi nó được triển khai. Một cách để thực hiện điều này là xem xét tất cả các giao dịch của hợp đồng, tuy nhiên điều đó là chưa đủ, vì các lời gọi (message call) giữa các hợp đồng không được ghi lại trên blockchain. Hơn nữa, nó cũng chỉ hiển thị các tham số đầu vào, không phải là những thay đổi trạng thái. Ngoài việc theo dõi hoạt động của hợp đồng ra, các sự kiện có thể được sử dụng để tương tác với giao diện người dùng.

```javascript
contract Charity {
    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

Ở trên, hợp đồng Game sẽ thực hiện lời gọi (internal call) đến Charity.donate(). Giao dịch này sẽ không xuất hiện trong danh sách các giao dịch bên ngoài (external transaction) của hợp đồng Charity mà có trong danh sách các giao dịch nội bộ (internal transaction).

Sự kiện là cách thuận tiện để ghi lại một điều gì đó đã xảy ra trong hợp đồng. Các sự kiện được phát ra (emit) được lưu lại trong blockchain cùng với dữ liệu khác của hợp đồng. Đây là một cải tiến cho ví dụ ở trên, chúng ta sử dụng sự kiện để ghi lại lịch sử quyên góp của Hội từ thiện.

```javascript
contract Charity {
    // define event
    event LogDonate(uint _amount);

    mapping(address => uint) balances;

    function donate() payable public {
        balances[msg.sender] += msg.value;
        // emit event
        emit LogDonate(msg.value);
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

Tất cả các giao dịch gọi hàm donate của hợp đồng Charity, dù trực tiếp hay không, sẽ hiển thị trong danh sách sự kiện của hợp đồng đó cùng với số tiền quyên góp.

### Sự phức tạp của ngôn từ (Lườm rau gắp thịt)

Khi bạn định nghĩa tên một hàm, nếu nó trùng với tên các hàm có sẵn của Solidity. Nó sẽ ghi đè (override) hàm mặc định nhưng nó sẽ gây hiểu nhầm cho người đọc với ý nghĩa của đoạn mã.

```javascript
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```

```javascript
contract FakingItsOwnDeath {
    function selfdestruct(address x) internal constant {}
}

contract SelfDestructive is FakingItsOwnDeath {
    function die() public {
        selfdestruct(address(0x0));
    }
}
```

Ở ví dụ thứ nhất, hàm revert() được gọi là hàm revert của hợp đồng PretendingToRevert thay vì hàm revert() mặc định. Do đó, khi gọi hàm someThingBad() của Example thì nó vẫn hoạt động bình thường.

Tương tự ở ví dụ thứ hai, không có hợp đồng nào bị hủy cả khi gọi đến hàm die() của SelfDestructive.

### Tránh sử dụng tx.origin

tx.origin chỉ có thể là tài khoản người dùng, không thể là tài khoản hợp đồng.
msg.sender có thể là tài khoản người dùng hoặc tài khoản hợp đồng.

Ví dụ các lời gọi theo chuỗi như sau: A->B->C->D, một hàm của hợp đồng D được gọi thì msg.sender là địa chỉ của C còn tx.origin là địa chỉ của A.

```javascript
contract MyContract {

    address owner;

    function MyContract() public {
        owner = msg.sender;
    }

    function sendTo(address receiver, uint amount) public {
        require(tx.origin == owner);
        receiver.transfer(amount);
    }

}

contract AttackingContract {

    MyContract myContract;
    address attacker;

    function AttackingContract(address myContractAddress) public {
        myContract = MyContract(myContractAddress);
        attacker = msg.sender;
    }

    function() public {
        myContract.sendTo(attacker, msg.sender.balance);
    }

}
```

Đoạn mã ở trên tận dụng đặc điểm của tx.orign để chuyển tiền từ hợp đồng MyContract vào tài khoản của kẻ xấu bằng cách viết hợp đồng AttackingContract và gọi đến hàm sendTo trong MyContract.

Khả năng trong tương lai, tx.origin sẽ bị loại bỏ khỏi nền tảng Ethereum. Chính nhà đồng sáng lập Ethereum Vatalik Buterin cho rằng tx.orgin không có ý nghĩa để có thể sử dụng trong hợp đồng thông minh.

Điều đáng nói là bằng cách sử dụng tx.origin, bạn sẽ hạn chế khả năng tương tác giữa các hợp đồng vì hợp đồng sử dụng tx.origin không thể được sử dụng bởi một hợp đồng khác vì tài khoản hợp đồng không thể là tx.origin.

### Nhãn thời gian (timestamp)

Có ba điều cần lưu ý khi sử dụng nhãn thời gian để thực hiện các chức năng quan trọng trong hợp đồng thông minh, đặc biệt là khi các hành động liên có quan đến việc chuyển tiền.

#### Thao tác với nhãn thời gian

Cần lưu ý rằng nhãn thời gian của một block có thể được tác động bởi thợ đào. Chúng ta cùng xem xét ví dụ sau đây:

```javascript
uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    //get the best seed for randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));

    return uint256((h / x)) % Max + 1; //random number between 1 and Max
}
```

Khi hợp đồng sử dụng nhãn thời gian để tạo số ngẫu nhiên, người thợ đào thực sự có thể đóng nhãn thời gian trong vòng 15 giây khi block đang được xác thực, cho phép người thợ đào có thể tính toán trước các tùy chọn. Nhãn thời gian không phải là ngẫu nhiên và không nên được sử dụng trong bối cảnh đó.

#### Quy tắc 15 giây

Trong Yellow Paper của Ethereum không mô tả về số lượng block có thể tạo ra trong một khoảng thời gian nhất định, nhưng nó đề cập rằng mỗi nhãn thời gian của block con phải lớn hơn nhãn thời gian của block cha mẹ. Các giao thức Ethereum trên Geth và Parity đều từ chối các block với nhãn thời gian lớn hơn 15 giây so với cha mẹ nó. Do đó, một nguyên tắc nhỏ trong việc đánh giá việc sử dụng nhãn thời gian là:

**Nếu sự kiện hợp đồng bạn triển khai có thể thay đổi trong 15 giây và duy trì tính toàn vẹn, thì việc sử dụng block.timestamp là an toàn.**

#### Tránh sử dụng block.number như là nhãn thời gian

Có thể ước tính một khoảng thời gian bằng cách sử dụng thuộc tính block.number và [thời gian khối trung bình](https://etherscan.io/chart/blocktime), tuy nhiên đây không phải là cách hay vì thời gian một block mới được tạo mới (block times) có thể thay đổi (ví dụ như việc xảy ra [fork reorganisations](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/) hoặc thay đổi [difficulty bomb](https://github.com/ethereum/EIPs/issues/649)).

### Hãy thận trọng khi sử dụng tính năng đa kế thừa

Khi sử dụng đa kế thừa trong Solidity, điều quan trọng là phải hiểu cách nó hoạt động như thế nào.

```javascript
contract Final {
    uint public a;
    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;

    function B(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;

    function C(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 5;
    }
}

contract A is B, C {
  function A() public B(3) C(5) {
      setFee();
  }
}
```

Khi một hợp đồng được triển khai, trình biên dịch sẽ tuyến tính hóa sự kế thừa từ phải sang trái (sau từ khóa ```is``` là danh sách các hợp đồng cha mẹ được liệt kê).


#### Final <- B <- C <- A

Hàm khởi tạo của hợp đồng A sẽ trả về 5, vì C là gần A nhất theo sự tuyết tính hóa từ phải qua trái. 

Để biết thêm về bảo mật và kế thừa, hãy xem [bài viết này](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/) 

Để giúp đóng góp, Github của Solidity có một [dự án](https://github.com/ethereum/solidity/projects/9#card-8027020) với tất cả các vấn đề liên quan đến thừa kế.

### Sử dụng interface thay vì address

Khi một hàm có tham số đầu vào là địa chỉ của một hợp đồng, tốt hơn là nên truyền vào interface hoặc một tham chiếu đến hợp đồng đó thay vì truyền vào địa chỉ của hợp đồng.
```javascript
contract Validator {
    function validate(uint) external returns(bool);
}

contract TypeSafeAuction {
    // good
    function validateBet(Validator _validator, uint _value) internal returns(bool) {
        bool valid = _validator.validate(_value);
        return valid;
    }
}

contract TypeUnsafeAuction {
    // bad
    function validateBet(address _addr, uint _value) internal returns(bool) {
        Validator validator = Validator(_addr);
        bool valid = validator.validate(_value);
        return valid;
    }
}
```

Những lợi ích của việc sử dụng hợp đồng TypeSafeAuction ở trên có thể được nhìn thấy từ ví dụ dưới đây. Nếu hàm validateBet() có tham số đầu vào là địa chỉ của hợp đồng, hoặc tham chiếu của một hợp đồng không phải là TypeSafeAuction thì trình biên dịch ném ra lỗi.

```javascript
contract NonValidator{}

contract Auction is TypeSafeAuction {
    NonValidator nonValidator;

    function bet(uint _value) {
        bool valid = validateBet(nonValidator, _value); // TypeError: Invalid type for argument in function call.
                                                        // Invalid implicit conversion from contract NonValidator
                                                        // to contract Validator requested.
    }
}
```

### Tránh sử dụng extcodesize để kiểm tra tài khoản người dùng (Externally Owned Accounts)

Modifier dưới đây có chức năng kiểm tra xem message gọi đến là tài khoản hợp đồng hay tài khoản người dùng.

```javascript
// bad
modifier isNotContract(address _a) {
  uint size;
  assembly {
    size := extcodesize(_a)
  }
    require(size == 0);
     _;
}
```

Ý tưởng rất đơn giản: nếu một địa chỉ có chứa mã nguồn, đó không phải là tài khoản người dùng mà là tài khoản hợp đồng. Tuy nhiên, một hợp đồng sẽ chưa bao gồm mã nguồn trong quá trình khởi tạo. Điều này có nghĩa là trong khi hàm contructor của hợp đồng đang được thực hiện, nó có thể thực hiện các lời gọi đến các hợp đồng khác với extcodesize trả về 0. Dưới đây là một ví dụ để làm rõ hơn.

```javascript
contract OnlyForEOA {    
    uint public flag;

    // bad
    modifier isNotContract(address _a){
        uint len;
        assembly { len := extcodesize(_a) }
        require(len == 0);
        _;
    }

    function setFlag(uint i) public isNotContract(msg.sender){
        flag = i;
    }
}

contract FakeEOA {
    constructor(address _a) public {
        OnlyForEOA c = OnlyForEOA(_a);
        c.setFlag(1);
    }
}
```

Đoạn mã trong hàm contructor của FakeEOA gọi đến hàm setFlag của OnlyForEOA, do hàm constructor của hợp đồng FakeEOA chưa được thực hiện xong, nên extcodesize của nó sẽ trả về 0 và vượt qua được bộ lọc của modifier isNotContract từ đó thay đổi giá trị flag trong OnlyForEOA một cách trái phép.

**Cảnh báo***: 
Nếu mục tiêu của bạn là ngăn chặn các hợp đồng khác có thể gọi đến hợp đồng của bạn, thì việc dùng extcodesize có lẽ cũng là tương đối tốt. Một cách tiếp cận khác là kiểm tra giá trị của (tx.origin == msg.sender), mặc dù điều này cũng có nhược điểm.

### Cẩn thận với phép chia cho 0 (Sodility <0,4)

Trước phiên bản 0.4, Solidity [trả về 0](https://github.com/ethereum/solidity/issues/670) và không ném ngoại lệ khi một số được chia cho 0. Đảm bảo bạn đang chạy phiên bản solidity từ 0.4 trở lên.

### Cách đặt tên function và event (Solidity <0.4.21)

Viết in hoa chữ đầu tiên tên của event, để ngăn ngừa rủi ro nhầm lẫn giữa các function và event. Đối với các function, luôn luôn bắt đầu bằng một chữ cái viết thường, ngoại trừ hàm khởi tạo (constructor).

## Bài viết được dịch một phần từ tài liệu "Smart contract best practice" của Consensys 

https://consensys.github.io/smart-contract-best-practices/recommendations/

# Hiểu về cách phương thức phổ biến tấn công hợp đồng thông minh 

Dưới đây là các lỗ hổng đã biết để có thể tấn công hợp đồng thông minh. Nó sẽ giúp chúng ta cẩn trọng hơn trong việc viết mã để tránh các lỗ hổng phổ biến này.

## Reentrancy

Một trong những mối nguy hiểm lớn từ gọi đến các hợp đồng bên ngoài là chúng có thể chiếm quyền điều khiển và thực hiện các thay đổi không mong muốn. Loại lỗi này có thể có nhiều biến thể và cả hai lỗi lớn dẫn vụ DAO hack nổi tiếng đều là các lỗi thuộc loại này.

### Reentrancy on single function

```javascript
// INSECURE
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call withdrawBalance again
    userBalances[msg.sender] = 0;
}
```

Vì số dư tài khoản của người dùng không gắn bằng 0 cho đến khi kết thúc function, hacker có thể lợi dụng điều này để rút tiền nhiều lần bằng cách gọi liên tục hàm withdrawBalance để ngăn cản hàm đó chạy đến câu lệnh ```userBalances[msg.sender] = 0; ```

> DAO (Decentralized Autonomous Organization). Mục đích mà nó hướng đến là mã hóa các quy tắc và bộ máy ra điều hành của một các tổ chức, loại bỏ vai trò của tài liệu và con người trong quá trình quản lý, tạo ra một cấu trúc với sự kiểm soát phi tập trung.
> 
> Vào ngày 17 tháng 6 năm 2016, DAO đã bị hack và 3,6 triệu Ether (50 triệu đô la) đã bị đánh cắp bằng cách khai thác lỗ hổng Reetrancy.
>
> Ethereum Foundation đã phát hành một bản cập nhật quan trọng để khôi phục lại trạng thái trước vụ hack. Điều này dẫn đến việc Ethereum được chia thành Ethereum Classic và Ethereum.

Trong ví dụ trên, cách giảm thiểu tối đa rủi ro là sử dụng send () thay vì call.value (). Điều này sẽ hạn chế bất kỳ mã bên ngoài nào được thực thi.

Tuy nhiên, nếu bạn không thể tránh các lời gọi từ bên ngoài, thì cách đơn giản để ngăn chặn cuộc tấn công này là đảm bảo bạn không gọi thực hiện lời gọi đến bên ngoài trước khi các đoạn mã internal được thực hiện xong.

```javascript
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    require(msg.sender.call.value(amountToWithdraw)()); // The user's balance is already 0, so future invocations won't withdraw anything
}
```

### Cross-function Reentrancy

Kẻ tấn công cũng có thể thực hiện một cuộc tấn công bằng cách sử dụng hai hàm khác nhau có cùng trạng thái.

```javascript
// INSECURE
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call transfer()
    userBalances[msg.sender] = 0;
}
```

Trong trường hợp này, kẻ tấn công sẽ gọi hàm transfer() khi đoạn mã khai thác của chúng đang thực hiện external call đến hàm withdrawBalance(). Vì khi đó số dư của kẻ tấn công chưa được gán bằng 0, nên chúng có thể chuyển mã token mặc dù chúng đã nhận được tiền từ hàm withdrawBalance. Lỗ hổng này cũng được sử dụng trong cuộc tấn công vào DAO.

Lưu ý rằng trong ví dụ này, cả hai hàm đều cùng thuộc một hợp đồng. Tuy nhiên, cùng một lỗi có thể xảy ra trên nhiều hợp đồng, nếu các hợp đồng đó chia sẻ trạng thái.

### Cạm bẫy trong các giải pháp Reentrancy

Vì reentrancy có thể xảy ra trên một chuỗi các hàm, thậm chí là các hợp, bất kỳ giải pháp nào nhằm ngăn chặn reentrancy chỉ với một hàm là không đủ.

Thay vào đó, chúng tôi khuyến nghị nên chạy các câu lệnh trong hợp đồng (internal work)  (ví dụ: thay đổi trạng thái các biến, mapping) trước việc thực hiện các lời gọi hàm bên ngoài (external call). Quy tắc này, nếu được tuân thủ cẩn thận, sẽ phòng chống được các lỗ hổng do reentrancy có thể gây ra. Tuy nhiên, bạn không chỉ cần tránh các lời gọi từ bên ngoài được thực thi quá sớm mà còn nên tránh việc các hàm bên trong hợp đồng gọi đến các hàm khác sớm (nên để câu lệnh thực thi sau khi các đoạn mã logic khác trong thân hàm đã chạy qua). Ví dụ sau đây minh chứng cho việc viết mã không an toàn.

```javascript
// INSECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    rewardsForA[recipient] += 100;
    withdrawReward(recipient); // At this point, the caller will be able to execute getFirstWithdrawalBonus again.
    claimedBonus[recipient] = true;
}
```

Mặc dù hàm getFirstWithdrawalBonus() không gọi từ hợp đồng bên ngoài, nhưng đoạn mã withdrawReward(recipient) gọi đến hàm withdrawReward đã tạo ra lỗ hổng mà kẻ xấu có thể khai thác. Do đó, phải xem lời gọi đến withdrawReward trong hàm getFirtWithdrawBonus() là không đáng tin cậy.

```javascript
// SECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdrawReward(recipient); // claimedBonus has been set to true, so reentry is impossible
}
```

Ngoài việc chuyển dòng code gọi đến hàm withdrawReward xuống cuối cùng trong hàm GetFirstWithdrawalBonus, chúng ta còn thay đổi tên hàm để đánh dấu nó là không đáng tin cậy.

Một giải pháp khác được đề xuất là [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion). Điều này cho phép bạn "khóa" một số trạng thái để chủ sở hữu khóa mới có thể thay đổi trạng thái. Một ví dụ đơn giản có thể trông như thế này:

```javascript
// Note: This is a rudimentary example, and mutexes are particularly useful where there is substantial logic and/or shared state
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() payable public returns (bool) {
    require(!lockBalances);
    lockBalances = true;
    balances[msg.sender] += msg.value;
    lockBalances = false;
    return true;
}

function withdraw(uint amount) payable public returns (bool) {
    require(!lockBalances && amount > 0 && balances[msg.sender] >= amount);
    lockBalances = true;

    if (msg.sender.call(amount)()) { // Normally insecure, but the mutex saves it
      balances[msg.sender] -= amount;
    }

    lockBalances = false;
    return true;
}
```

Nếu người dùng cố gắng gọi hàm withdraw() một lần nữa trước khi câu lệnh ```msg.sender.call(amount)()``` kết thúc, khóa sẽ ngăn cản điều đó. Đây có thể là một giải pháp hiệu quả, nhưng nó trở nên khó khăn khi bạn có nhiều hợp đồng liên kết với nhau. Sau đây ví dụ minh chứng cho nhận định trên:

```javascript
// INSECURE
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == address(0));
        lockHolder = msg.sender;
    }

    function releaseLock() {
        require(msg.sender == lockHolder);
        lockHolder = address(0);
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

Kẻ tấn công có thể gọi hàm getLock(), và sẽ không bao giờ gọi đến releaseLock(). Nếu điều này xảy ra, thì hợp đồng sẽ bị khóa vĩnh viễn và không thể thay đổi trạng thái của lockHolder. Nếu bạn sử dụng mutexes để phòng chống reentrancy, bạn sẽ cần chắc chắn rằng sẽ không có trường hợp bị khóa cứng như đoạn mã ở trên. (Có những nguy cơ tiềm tàng khác khi lập trình với mutexes, chẳng hạn như deadlocks và livelocks. 

## Front-Running (AKA Transaction-Ordering Dependence)

Ở phần trên chúng ta đã đề cập đến thức khai thác và phòng chống lỗ hổng reentrancy. Bây giờ chúng ta sẽ cùng tìm hiểu một kiểu tấn công khác trong Blockchain dựa vào đặc điểm rằng: thứ tự của các giao dịch (ví du: trong một block) có thể bị kiểm soát.

Since a transaction is in the mempool for a short while, one can know what actions will occur before it is included in a block. This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against this is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another way to use a pre-commit scheme (“I’m going to submit the details later”).

Một giao dịch trước khi được xác minh thì nó nằm trong mempool một thời gian ngắn, người ta có thể biết được những gì xảy ra với giao dịch trước khi nó được đưa vào một block. Điều này có thể gây rắc rối cho những sàn giao dịch phi tập trung, nơi mọi người có thể nhìn thấy tất cả các giao dịch. Để ngăn cản điều này thực sự rất khó, vì nó sẽ đi vào chính hợp đồng cụ thể. Ví dụ, tại các thị trường, sẽ tốt hơn nếu thực hiện đấu giá hàng loạt (điều này cũng bảo vệ chống lại việc phân tích các giao dịch tần suất cao). Một cách khác để sử dụng sơ đồ pre-commit (Chúng tôi sẽ đi vào chi tiết ở phần sau).

## Tràn số nguyên (underfow và overflow)

Chúng ta hãy còn xem đoạn mã dưới đây:

```javascript
mapping (address => uint256) public balanceOf;

// INSECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance */
    require(balanceOf[msg.sender] >= _value);
    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// SECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance and for overflows */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

Nếu giá trị ```balanceOf[_to]``` đạt đến giá trị lớn nhất có thể (2 ^ 256), nếu tăng ```balanceOf[_to]```, nó sẽ quay vòng về giá trị 0. Khi thực hiện phép tính với các biến, hãy suy nghĩ xem về việc liệu giá trị uint có cơ hội tiếp cận một số lớn như vậy hay không. Hãy xem rằng các biến uint có thể thay đổi giá trị bởi những ai. Nếu bất kỳ người dùng nào có thể gọi các hàm cập nhật giá trị uint, thì rất là nguy hiểm. Sẽ an toàn hơn nếu chỉ có quản trị viên có quyền truy cập để thay đổi trạng thái của biến. Nếu người dùng chỉ có thể tăng thêm 1 lần một, có lẽ bạn cũng an toàn vì khó có cách nào khả thi để đạt đến giới hạn số  nguyên dương.

Tương tự với underflow. Nếu một uint bị giảm giá trị đến một số âm nhỏ hơn 0, nó sẽ quay vòng lại số lớn nhất của kiểu dữ liệu.

Một giải pháp đơn giản để giảm thiểu lỗi tràn số là sử dụng thư viện [SafeMath.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/math/SafeMath.sol) cho các hàm số học.

### Tìm hiểu chi tiết hơn về underflow

Bài dự thi của Doug Hoyte trong cuộc thi về sự vững chắc trong năm 2017 đã nhận được một đề cập đáng trân trọng. Mục này rất thú vị, bởi vì nó làm tăng mối lo ngại về việc dòng chảy giống như C có thể ảnh hưởng đến việc lưu trữ Solidity như thế nào. Đây là một phiên bản đơn giản hóa:

```javascript
contract UnderflowManipulation {
    address public owner;
    uint256 public manipulateMe = 10;
    function UnderflowManipulation() {
        owner = msg.sender;
    }

    uint[] public bonusCodes;

    function pushBonusCode(uint code) {
        bonusCodes.push(code);
    }

    function popBonusCode()  {
        require(bonusCodes.length >=0);  // this is a tautology
        bonusCodes.length--; // an underflow can be caused here
    }

    function modifyBonusCode(uint index, uint update)  {
        require(index < bonusCodes.length);
        bonusCodes[index] = update; // write to any index less than bonusCodes.length
    }

}
```

Nói chung, vị trí của thao tác biến có thể bị ảnh hưởng mà không đi qua keccak256, điều này là không khả thi. Tuy nhiên, vì các mảng động được lưu trữ tuần tự, nếu một tác nhân độc hại muốn thay đổi thao tác, tất cả những gì họ cần làm là:
* Gọi ```popBonusCode``` để underflow (Lưu ý: phương thức Array.pop () đã được thêm vào trong Solidity 0.5.0)
* Tính toán vị trí lưu trữ của thao ```manipulateMe```
* Sửa đổi và cập nhật giá trị của thao tác bằng cách sử dụng ```notifyBonusCode```











