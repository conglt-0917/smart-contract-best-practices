# Solidity Recommendations

Phần này trình bày một số mẫu thiết kế thường được sử dụng khi viết hợp đồng thông minh.

## Khuyến nghị cụ thể về giao thức

### Lời gọi từ bên ngoài (External Calls)

#### Hãy thật cẩn trọng khi sử dụng external calls

Các message gọi đến hợp đồng không đáng tin cậy có thể gây ra một số rủi ro hoặc lỗi không mong muốn. Các lời gọi bên ngoài có thể thực thi mã độc trong hợp đồng đó hoặc bất kỳ hợp đồng nào khác mà nó phụ thuộc vào. Như vậy, mọi lời gọi bên ngoài nên được xem là ẩn chứa rủi ro bảo mật. Trong trường hợp bất khả kháng, hãy sử dụng các đề xuất của chúng tôi ở bên dưới để giảm thiểu rủi ro có thể xảy ra.

#### Đánh dấu các hợp đồng không đáng tin cậy

Khi tương tác với các lời gọi bên ngoài, tên các biến, phương thức và các interface nên được đặt sao cho thể hiện được việc tương tác với các lời gọi từ bên ngoài có an toàn hay không ?. Điều này áp dụng cho các hàm mà nó có thể được gọi các hợp đồng bên ngoài.

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

Nếu bạn đang thực hiện lời gọi đến một hợp đồng bên ngoài không đáng tin cậy, hãy tránh thay đổi trạng thái sau lời gọi. Nguyên tắc này đôi khi được gọi với các tên checks-effects-interactions pattern

#### Sự khác nhau giữa send (), transfer () và call.value ()

Khi thực hiện một giao dịch từ hợp đồng thông minh, cần biết sự giống và khác giữa **someAddress.send()**, **someAddress.transfer()**, **someAddress.call().value()**

- someAddress.send () và someAddress.transfer () được coi là an toàn chống lại reentrancy. Mặc dù từ các phương thức này vẫn có thể kích hoạt mã thực thi, nhưng lời gọi chỉ được giới hạn 2.300 gas, nó chỉ đủ để ghi lại một sự kiện thay vì chạy một đoạn mã khai thác.

- x.transfer(y) tương đương với lệnh x.send (y), nó sẽ tự động revert nếu giao dịch thất bại.

- Khác với someAddress.send () và someAddress.transfer (), someAddress.call.value (y) không có giới hạn gas cho lời gọi và do đó hacker có thể thực thi lời gọi đến một đoạn mã độc nhằm mục đích xấu.Do đó, nó không an toàn chống lại reentrancy.

Sử dụng send () hoặc transfer () sẽ ngăn chặn reentrancy nhưng nó sẽ không thích hợp với hợp đồng mà fallback function yêu cầu hơn 2.300 gas. Chúng ta cũng có thể sử dụng someAddress.call.value(ethAmount) .gas(gasAmount) để giới hạn lương gas cho lời gọi.

#### Xử lý lỗi trong các lời gọi bên ngoài

Solidity cung cấp các phương thức gọi mức thấp (low level) : address.call(), address.callcode(), address.delegatecall() và address.send(). Các phương thức cấp thấp này không bao giờ ném ra ngoại lệ (throw an exception), nhưng sẽ trả về false nếu lời gọi gặp phải ngoại lệ. Mặt khác, các lời gọi hợp đồng (contract calls)

(ví dụ như ExternalContract.doSomething()) sẽ tự động ném ra một ngoại lệ và báo lỗi.

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

Các lời gọi từ bên ngoài có thể thất bại vô tình hoặc cố ý. Để giảm thiểu rủi ro từ các lỗi đó gây ra, tốt hơn hết là chia từng lời gọi bên ngoài thành giao dịch có thể được khởi tạo bởi người nhận lời gọi. Điều này đặc biệt phù hợp với các giao dịch thanh toán, trong đó cho phép người dùng rút tiền sẽ tốt hơn là tự động chuyển tiền cho họ. (Điều này cũng làm giảm khả năng xảy ra sự cố với gasLimit.) Tránh việc thực hiện cùng một lúc nhiều hàm transfer() trong một giao dịch.

```javascript

// bad
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

#### Đừng dùng delegatecall với đoạn mã không được tin cậy

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

**\*Lưu ý**: Đừng cho rằng các hợp đồng khi được khởi tạo có số dư bằng 0. Một kẻ tấn công có thể gửi ether đến địa chỉ của hợp đồng trước khi nó được khởi tạo. [Xem vấn đề 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61) để biết thêm chi tiết.\*

### Ether có thể được gửi đến bất kỳ hợp đồng thông minh nào

Coi chừng mã hóa một bất biến kiểm tra chặt chẽ sự cân bằng của hợp đồng.

Kẻ tấn công có thể gửi ether đến bất kỳ tài khoản nào và điều này không thể ngăn chặn được (ngay cả với fallback function gồm câu lệnh revert).

Kẻ tấn công có thể làm điều này bằng cách tạo ra một hợp đồng, gửi cho nó 1 wei và hàm selfdestruct(victimAddress), ở đây victimAddress là tài cần gửi ether vào. Không có mã nào được gọi trong hợp đồng có địa chỉ victimAddress, vì vậy nó không thể ngăn chặn.

### Hãy nhớ rằng Ethereum là mạng public blockchain, mọi dữ liệu trên các block đều được công khai

Nhiều ứng dụng yêu cầu dữ liệu được gửi phải ở chế độ riêng tư cho đến một lúc nào đó để hoạt động. Các trò chơi (ví dụ: oản tù tì) và việc đấu giá kín là hai loại ví dụ chính. Nếu bạn đang xây dựng một ứng dụng mà sự riêng tư là một vấn đề, hãy đảm bảo bạn tránh yêu cầu người dùng công khai thông tin quá sớm. Chiến lược tốt nhất là chia thành các giai đoạn riêng biệt: đầu tiên thì sử dụng hàm băm của các giá trị và trong giai đoạn tiếp theo thì tiết lộ các giá trị.

Ví dụ:

- Trong trò chơi oản tù tì, yêu cầu cả hai người chơi gửi giá trị băm của kéo, đá hay giấy (do người chơi quyết định), sau đó trò chơi yêu cầu cả hai người chơi gửi kết quả mình lựa chọn. Tiếp đó so sánh giá trị băm, nếu khớp thì hợp lệ, trò chơi sẽ phân thắng hòa hay thua dựa trên kết quả chọn của 2 người chơi.

- Trong phiên đấu giá kín, yêu cầu người đấu giá gửi giá trị băm mức giá mà họ chọn trong giai đoạn ban đầu (cùng với khoản tiền gửi lớn hơn giá trị giá thầu của họ), sau đó gửi giá trị đấu giá của họ trong giai đoạn thứ hai.

- Khi phát triển một ứng dụng mang tính ngẫu nhiên, thứ tự phải luôn là (1) người chơi submit, (2) số ngẫu nhiên được tạo, (3) người chơi hoàn thành giao dịch. Phương thức mà các số ngẫu nhiên được tạo ra là cả một lĩnh vực nghiên cứu, các giải pháp tốt nhất hiện tại bao gồm có tiêu đề block header Bitcoin (được xác minh thông qua http://btcrelay.org), các cơ chế hash-commit-reveal (tức là một bên tạo ra một số, xuất bản hàm băm của nó để "cam kết" và sau đó tiết lộ giá trị sau), cùng với đó là [RANDAO](https://github.com/randao/randao). Vì Ethereum là một giao thức xác định, không có biến nào trong giao thức có thể được sử dụng như một số ngẫu nhiên không thể đoán trước. Ngoài ra, hãy lưu ý rằng thợ đào (miner) trong một chừng mực nào đó kiểm soát giá trị block.blockhash().

-

### Cảnh giác với khả năng một số người tham gia có thể "drop offline" và không quay lại

Không thực hiện các quy trình hoàn trả hoặc yêu cầu phụ thuộc vào một bên cụ thể thực hiện một hành động cụ thể mà không có cách nào khác để rút tiền. Ví dụ, trong trò chơi oản tù tì, một lỗi phổ biến là kết thúc ván đấu cho đến khi cả hai người chơi gửi lựa chọn của họ.Tuy nhiên, một người chơi có thể không bao giờ gửi lựa chọn của họ - thực tế, nếu một người chơi thấy động thái được tiết lộ từ người chơi khác và xác định rằng họ đã thua, họ không có lý do gì để tự gửi kết quả. Khi gặp các tình huống như vậy thì , (1) cung cấp một cách để tránh những người tham gia không tham gia, có thể giới hạn thời gian và (2) xem xét thêm một lợi ích bổ sung cho những người tham gia khi gửi kết quả trong tất cả các tình huống họ thắng hoặc thua.

### Trường hợp đổi dấu số âm bé nhất

Solidity cung cấp một số kiểu dữ liệu số nguyên. Giống như trong hầu hết các ngôn ngữ lập trình khác, trong Solidity, một số nguyên được N bit có thể biểu thị các giá trị từ -2 ^ (N-1) đến 2 ^ (N-1) -1. Điều này có nghĩa là không có giá trị dương mà có trị tuyệt đối là |MIN_INT|. - MIN_INT sẽ bằng MIN_INT

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

## Các khuyến nghị cụ thể

Các khuyến nghị sau đây dành riêng cho Solidity, nhưng cũng có thể là hướng dẫn để phát triển hợp đồng thông minh bằng các ngôn ngữ khác.

### Thực thi bất biến với assert ()

Assert khẳng định kích hoạt khi một xác nhận thất bại - chẳng hạn như một thuộc tính bất biến thay đổi. Ví dụ:
trong hợp đồng phát hành token, số lượng token có thể được cố định. Bạn có thể xác minh với assert. Assert thường được kết hợp với các kỹ thuật khác, chẳng hạn như tạm dừng hợp đồng và cho phép nâng cấp. (Nếu không, bạn có thể sẽ bị mắc kẹt, với assert luôn luôn thất bại.)

```javascript
contract Token {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        totalSupply += msg.value;
        assert(this.balance >= totalSupply);
    }
}
```

### Sử dụng assert(), revert() và require() đúng cách

Các assert và require được sử dụng để kiểm tra các điều kiện và ném ra một ngoại lệ nếu điều kiện không được đáp ứng.

Hàm **assert** chỉ nên được sử dụng để kiểm tra các lỗi bên trong và để kiểm tra các biến hằng.

Hàm **require** nền được dùng để đảm bảo các điều kiện hợp lệ, chẳng hạn như biến đầu vào, biến trạng thái của hợp đồng hoặc để xác thực giá trị trả về từ các lời gọi đến hợp đồng bên ngoài.

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

### Chỉ sử dụng công cụ sửa đổi cho các xác nhận

Mã bên trong modifier được thực thi trước khi chạy mã bên trong hàm.Do đó, bất kỳ thay đổi trạng thái hoặc lời gọi bên ngoài nào được tạo ra bởi đoạn mã trong modifier cũng sẽ vi phạm thiết kế [Checks-Effects-Interactions](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) mà chúng tôi đã đề ra. Ví dụ dưới đây, một lời gọi bên ngoài hợp đồng được chèn trong modifier có thể dẫn đến lỗ hổng reentrancy.

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

**Lưu ý**: Sử dụng modifier để thay các câu lệnh kiểm tra điều kiện bên trong thân của hàm. Điều này làm cho mã nguồn hợp đồng thông minh của bạn dễ đọc hơn và kiểm toán hơn.

### Hãy cẩn thận với việc làm tròn số trong phép chia

Tất cả các phép chia số nguyên được làm tròn bằng cách lấy số nguyên gần nhất. Nếu bạn cần độ chính xác cao hơn, hãy cân nhắc lưu trữ cả tử và mẫu số, hoặc số nhân.

(Trong tương lai, Solidity sẽ có [fixed_point type](https://solidity.readthedocs.io/en/develop/types.html#fixed-point-numbers), điều này sẽ khiến vấn đề dễ dàng hơn.)

```javascript
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer
```

Sử dụng số nhân sẽ ngăn việc làm tròn xuống, số nhân này cần được tính khi làm việc với x trong tương lai:

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

Cả interface và hợp đồng trừu tượng (abstract contract) đều cung cấp một có chung một tư tưởng là tùy chỉnh mã nguồn các function dựa trên prototype có sẵn. Interface , được giới thiệu trong phiên bản Solidity 0.4.11, tương tự như các abstract contract nhưng interface chỉ có các prototype của mà mà không chứa thân hàm. Interface cũng có những hạn chế như không thể truy cập vào storage hoặc kế thừa từ các interface khác, điều này thường làm cho các abstract contract có ưu thế hơn một chút. Ngoài ra, điều quan trọng cần lưu ý là nếu một hợp đồng kế thừa từ một abstract contract thì các hàm sẽ được thực thi bằng cách ghi đè (overriding), nếu không nó vẫn sẽ trừu tượng.

### Fallback functions

#### Giữ cho Fallback function đơn giản

[Fallback function](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) được gọi khi một hợp đồng được gửi một message không có tham số (hoặc gọi một hàm mà trong hợp đồng đó không tồn tại) và chỉ có tối đa 2.300 gas gọi bằng các hàm .send () hoặc .transfer (). Nếu bạn chỉ muốn nhận Ether fallback function bằng cách gọi .send () hoặc .transfer (), thì 2300 gas đủ để cho bạn kích hoạt một event. Nếu cần sử nhiều tính toán hơn thì có thể cấu hình lượng gas tối đa mà fallback function có thể sử dụng.

```javascript
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

### Kiểm tra độ dài dữ liệu trong fallback function

Fallback function không chỉ được dùng để nhận ether gửi vào hợp đồng (không có dữ liệu trong message) mà còn khi gọi hàm không có trong hợp đồng hoặc tham số không đúng. Do đó, kiểm tra độ dài data trước khi thực thi các mã trong fallback function nhằm tránh việc bị thực thi mã độc.

```javascript
// bad
function() payable { emit LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); emit LogDepositReceived(msg.sender); }
```

### Định nghĩa rõ ràng các hàm phải có thể nhận ether và các biến

Bắt đầu từ phiên bản Solidity 0.4.0, mọi hàm nhận ether phải có modifier **payable**, mặt khác, nếu lời gọi đến hàm payble có msg.value = 0 thì giao dịch sẽ bị revert ([trừ khi bắt buộc](https://consensys.github.io/smart-contract-best-practices/recommendations/#remember-that-ether-can-be-forcibly-sent-to-an-account)).

Khai báo các biến và đặc biệt là các tham số của hàm có dạng **address payable** , nếu bạn muốn dùng chức năng chuyển tiền. Bạn có thể sử dụng .transfer (..) và .send (..) trên **address payable**, không phải trên **address**. Bạn có thể sử dụng .call (..) cho cả **address payable** và **address**. Nhưng điều này không được khuyến khích.

**Lưu ý**: Modifier payable chỉ áp dụng cho các cuộc lời từ các hợp đồng bên ngoài. Nếu ta gọi một hàm non-payable trong hàm payable trên cùng một hợp đồng, thì hàm non-payable sẽ không thành công, mặc dù msg.value vẫn được đặt.

### Định nghĩa rõ ràng phạm vi truy cập của các hàm, các biến

Các hàm có bốn loại phạm vi truy cập là external, public, private, internal. . Đối với các biến, không thể định nghĩa phạm vi external. Định nghĩa đầy đủ, rõ ràng phạm vi truy cập của các biến, các hàm giúp dễ dàng nắm được được phạm vi của từng thành phần trong hợp đồng, tránh các lỗi không đáng có.

- Các hàm external là một phần chức năng của contract interface. Các hàm external hiểu quả hơn các hàm public khi tham số là các mảng dữ liệu lớn do hàm external sẽ tốn ít gas hơn.
- Các hàm public có thể được gọi từ bất cứ đâu, trong hợp đồng, hoặc từ một hợp đồng khác.
- Các hàm internal chỉ có thể được gọi từ bên trong hợp đồng hoặc các hợp đồng kế thừa
- Các hàm private chỉ có thể được gọi từ bên trong hợp đồng

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

```javascript
// bad
pragma solidity ^0.4.4;

// good
pragma solidity 0.4.4;
```

### Sử dụng các sự kiện để theo dõi hoạt động của hợp đồng

Có thể hữu ích khi có cách giám sát hoạt động của hợp đồng sau khi được triển khai. Một cách để thực hiện điều này là xem xét tất cả các giao dịch của hợp đồng, tuy nhiên điều đó có thể không đủ, vì các cuộc gọi tin nhắn giữa các hợp đồng không được ghi lại trong blockchain. Hơn nữa, nó chỉ hiển thị các tham số đầu vào, không phải là những thay đổi thực tế được thực hiện cho trạng thái. Ngoài ra các sự kiện có thể được sử dụng để kích hoạt các chức năng trong giao diện người dùng.

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
