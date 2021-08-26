# Capture the ether writeup
Capture the ether là một game tương tự Capture the Flag với target cụ thể hơn là các lỗi bảo mật trong các hợp đồng thông minh Ethereum.
Đây là writeup một số challenge của capture the ether trên [Website](https://capturetheether.com/)

## Lottery
### Guess the number [Link](https://capturetheether.com/challenges/lotteries/guess-the-number/)
Đây là bài đầu trong category này nên tương đối đơn giản. Chủ yếu là để người chơi làm quen với cú pháp solidity và các thư viện tương tác.

Challenge này sẽ complete nếu người chơi lấy hết được balance của thử thách thông qua function guess().

Người chơi chỉ cần gọi function guess() với n=42 tương ứng với initial answer và kèm theo value của yêu cầu là 1 ether. Qua các bước kiểm tra, contract sẽ gửi 2 ether về cho người chơi. Người chơi có thể lặp đi lặp lại các bước này cho tới khi balance của contract bằng 0 và hoàn thành thử thách.

### Guess the secret number [Link](https://capturetheether.com/challenges/lotteries/guess-the-secret-number/)
Thử thách thứ hai cũng tương tự challenge đầu, nhưng bây giờ answer không được lưu trực tiếp nữa. Và câu lệnh kiểm tra sẽ kiểm tra hash của input và cụ thể là keccak256. 

Bình thường để dò ngược giá trị gốc trước khi hash là một sự vô ích vì ta không biết được chiều dài và kích thước cụ thể của input và có nhiều input có cùng một kết quả hash. Tuy nhiên ở challenge này, input vào là uinit8 có miền giá trị từ 0-255 nên ta có thể tính thử 
hash của các giá trị này và so với answerhash.

Bài học kinh nghiệm:
1. Chú ý đến miền giá trị của các kiểu dữ liệu

### Guess the random number [Link](https://capturetheether.com/challenges/lotteries/guess-the-random-number/)

THử thách này cũng có bố cục giống hai thử thách ban đầu. Tuy nhiên lần này answer đã được tính bởi hash một giá trị động thông qua thời gian khởi tạo của Contract. Và đây là điều mà ta không thể tính toán được.

Sử dụng kinh nghiệm từ challenge vừa rồi, ta thấy answer được gán với kiểu dữ liệu là uint8 nên chắc chắn có giá trị từ 0-255 thôi. Nên ta có thể dùng gọi function guess với giá trị từ 0-255.

Đây chắc chắn là một cách giải hợp lệ. Nhưng ether ở đâu mà đoán nhiều được như thế, mỗi lần đoán mất 1 ether. để đoán hết từ 0-255 tốn kha khá mỗi lần. Xui thì mình sẽ tay trắng trước khi tìm được answer.

Sau một hồi tìm kiếm và tham khảo từ nhiều nguồn. Mình biết được là ta có thể đọc giá trị của các state variable trong contract một cách thoải mái. Dẫu cho state variable trong contract là public, private hay internal. Bởi vì tất cả trên blockchain đều công khai. Việc mình cần làm là truy cập vào storage tại vị trí tương ứng của state variable thôi. Sử dụng lib web3 mình có truy cập tới giá trị answer ở slot 0.

```
        const contractAddress = '0x77cE55586437aBB1c5Ba20024601A2e44D4152bF';
        let index=0;
        let result=await web3.eth.getStorageAt(contractAddress, index);
        console.log(result);
```

Biết được giá trị của answer mình chỉ cân gọi hàm guess() và truyền thông tin vào để lấy ether thôi. Kéo balance của contract về 0 mình có thể complete rồi.

Kiến thức thu được:
1. Tất đều là publish nếu lưu trữ trong blockchain, cho dù tầm vực là publish, private hay internal. Ta chỉ cần địa chỉ trong storage thì đều truy cập được hết.

## MATH
### Token Sale [Link](https://capturetheether.com/challenges/math/token-sale/)
Contract này hỗ trợ một số hàm mua bán token tuy nhiên ở hàm mua token có một lỗi overflow.

    uint256 constant PRICE_PER_TOKEN = 1 ether;
    
    function buy(uint256 numTokens) public payable {
        require(msg.value == numTokens * PRICE_PER_TOKEN);

        balanceOf[msg.sender] += numTokens;
    }
Dòng require(msg.value == numTokens * PRICE_PER_TOKEN). Có câu lệnh nhân nhưng sau đó không kiểm tra overflow.

Nếu nghĩ đơn gián PRICE_PER_TOKEN giá trị là 1 ether thì đúng là không cần kiểm trả thật vì vế phải bằng numTokens, làm sao mà overflow được. Tuy nhiên với solidity 1 ether tương đương với 10^18 wei đơn vị coin nhỏ nhất. Nên ta có thể nhập (2^256-1)/10^18 n (lấy cận trên) là giá trị của numTokens để biểu thức bên phải overflow.

Giá trị token mua: 115792089237316195423570985008687907853269984665640564039458.
Khi ta chỉ cần mua với giá (numToken-(2^256-1)/10^18) = 0.41599208687036004 ether. 
Sau khi mua dư token, ta có thể bán lại để balance contract về 0. Hàm isComplete sẽ trả về true.

Bài học rút ra từ challenge này:
1. Cẩn thận khi sử dụng các đơn vị chỉ số lượng coin, chúng được quy đổi về đơn vị nhỏ nhất wei khi lưu trữ và tính toán.
1. Cộng, trừ, nhân chia đều cần kiểm tra lại vì có thể dính overflow hoặc underflow.

# Token whale [Link](https://capturetheether.com/challenges/math/token-whale/)

Challenge này mô phỏng ERC20 với các chức năng chuyển token. Player khởi đầu balance sẽ là 1000 để solve được challenge này, player phải transfer để balance >=1000000.
Sau khi đọc source code thì ta thấy một chỗ bất hợp lý.
Trong quá trình gọi hàm trong hai hàm sau đây.

    function transferFrom(address from, address to, uint256 value) public {
        require(balanceOf[from] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);
        require(allowance[from][msg.sender] >= value);

        allowance[from][msg.sender] -= value;
        _transfer(to, value);
    }

    function _transfer(address to, uint256 value) internal {
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;

        emit Transfer(msg.sender, to, value);
    }
    
Hàm transferFrom cho phép chuyển tài khoản từ địa chỉ nguồn (from) tới đích (to). Khi chuyển qua hàm _transfer lại không truyển địa chỉ nguồn, mà sử dụng người gọi hàm (msg.sender) để trừ balance.
Có thể hiểu nôm na lỗi như như sau:
A là người trung gian thực thi chức năng chuyển tiền từ B đến C. Nhưng token gửi tới C lại trừ vào tài khoản của A thay vì tài khoản của B.

Cộng thêm balanceOf[msg.sender] không được kiểm tra underflow nếu ta trừ vào tài khoản có balanceOf tiệm cận 0. 

Kịch bản thực thi ở đây sẽ là:
1. Tạo một tài khoản thứ 2.
1. Gọi hàm approve để ủy quyền cho tài khoản 2 chuyển tiền thay tài khoản 1. 
1. Dùng tài khoản 2 chuyển token từ tài khoản 1 đến tài khoản một với value > 1 (Lúc này tài khoản 2 sẽ bị trừ với value tương ứng dẫn tới underflow).
1. Chuyển token từ tài khoản 2 tới tài khoản 1 đủ đề qua challenge.

Bài học kinh nghiệm:
1. Sử dụng các biến toàn cục một các thông minh. Nên truyền cụ thể các giá trị qua các hàm.
2. Vẫn là kiểm tra underflow và overflow sau khi tính toán.

### Retirement fund [Link](https://capturetheether.com/challenges/math/retirement-fund/

Thử thách này mô phỏng một cam kết gửi tiền. Nếu người gửi rút tiền ra sớm thì chỉ nhật được 90% giá trị và 10% phạt mình có thể rút. Nếu người gửi rút đủ tiền, balance của contract=0 thì challenge này hoàn thành.

Sau khi xem xét source của challenge ta có thể thấy có lỗi underflow ở function collectPenalty()

    function collectPenalty() public {
        require(msg.sender == beneficiary);

        uint256 withdrawn = startBalance - address(this).balance;

        // an early withdrawal occurred
        require(withdrawn > 0);

        // penalty is what's left
        msg.sender.transfer(address(this).balance);
    }
Cụ thể là với biến withdrawn khi mà nó được tính từ startBalance và balance hiện tại của contract. Nếu balance > startBalalance thì require sẽ được pass và ta có thể lấy toàn bộ balance hiện tại của contract.

Để tăng giá trị balance contract. Mình đa thử gửi trực tiếp coin tới contract nhưng không được. Cụ thể là do Contract này chưa hiện thực hàm fallback() với keyword là payable.

Giải thích một chút về hàm fallback của solidity. Khi người dùng sử dùng gọi các chức năng không có trong contract thì nó sẽ gọi hàm fallback(). Mặc định khi không hiện thực hàm này thì hàm sẽ không có keyword ``payable'' nghĩa là hàm sẽ raise exception khi nhận được coin gửi tới. Với exception này, contract sẽ gửi trả lại coin theo address gửi.

Để có thể gửi coin tới contract, mình đã tìm kiếm cách gửi với keyword: "force send ether to contract". Có hai cách đó là sử dụng một contract với hàm hủy và gửi coin với tới địa chỉ contract trước khi nó được khởi tạo [Link blog](https://medium.com/@alexsherbuck/two-ways-to-force-ether-into-a-contract-1543c1311c56). Ở đây thì mình chỉ có thể dùng cách thứ nhất. 

Code solidity của mình như sau :
```
pragma solidity ^0.7.3;

contract RetirementFundAttacker {
    constructor () payable {
        require(msg.value > 0);
        victimAddress="xxxxxx"
        selfdestruct(victimAddress);
    }
}
```

Khi contract này bị hủy, balance sẽ được gửi tới victiomAddress và exception do fallback() raise ra sẽ không thể gửi coin lại được. Vì địa chỉ contract này còn contract nào hoạt động đâu :3.

Sau khi thêm balance cho contract gốc thì balance > startBalance dẫn tới ta có thể collect hết được balance theo như hàm collectPenalty. Challenge đã complete.

Kiến thức thu được từ challenge:
1. Hàm fallback() sẽ được gọi khi không có chức năng nào tương ứng với yêu cầu của người gọi.
1. Không có keyword payable function sẽ raise excection khi người gọi truyền coin vào
1. Có thể force send coin tới cointract đang hoạt động bằng hàm selfdestruct().


### Mapping [Link](https://capturetheether.com/challenges/math/mapping/

Đây là một bài có source code khá đơn giản, người dùng có thể lưu các giá trị theo key vào mảng và kiểm tra giá trị của các key trong mảng. Challenge sẽ hoàn thành khi mà state variable isComplete là True.
```
pragma solidity ^0.4.21;

contract MappingChallenge {
    bool public isComplete;
    uint256[] map;
    
    function set(uint256 key, uint256 value) public {
        // Expand dynamic array as needed
        if (map.length <= key) {
            map.length = key + 1;
        }
        map[key] = value;
    }
    
    function get(uint256 key) public view returns (uint256) {
        return map[key];
    }
}
```
Mục tiêu của challenge này khá rõ ràng. Mình cần điều khiển giá trị key của hàm se() làm sao để có thể write vào biến isComplete.

Để kiểm soát được địa chỉ này, ta cần biết cách thức lưu trữ và ghi của các biến trong Contract. Theo mình tìm hiểu thì:
Các state variable sẽ được lưu trong các vị trí là slot, bắt đầu từ slot_0, mỗi slot sẽ có thể chứa 256 bit. Các kiểu dữ liệu cơ bản sẽ được lưu liên tiếp nhau, cho đến khi đầy slot thì mới chuyển tiếp qua slot tiếp theo.

Riêng với 2 kiểu dữ liệu là array và map sẽ có một chút đặc biệt. Chúng đều sẽ được lưu ở slot mới, slot này sẽ được array lưu chiều dài của array, còn map thì không sử dụng nó. Dữ liệu của array sẽ được lưu liên tiếp nhau  bắt đầu từ địa chỉ keccak256(slot\_number), các giá trị kéo theo sẽ được lưu ở  keccak256(slot\_number) + 1, keccak(slot\_number) + 2, ... ,  keccak256(slot\_number) + n. Còn map thì lưu data ở vị trí keccak(key+slot\_number).

Vị trí lưu cũng chính là vị trí mà ta có thể write vào
Địa chỉ mà ta có thể ghi vào với hàm set là: keccak256(1)+key (1 là vị trí slot của biến map).
Việc tiếp theo của ta là điều khiển key làm sao để có thể lưu vào địa chỉ 0 là đị chỉ lưu giá trị biến isComplete.

Cận trên của giá trị trong solidity là của uint256 là 2^256-1. Vậy nên nếu: keccak256(1)+key > 2^256-1 sẽ bị overflow và modulo lại theo 2^256.
=> Để địa chỉ ghi vào là 0 thì key=2^256-keccak256(1)
Value ta truyền vào là 1 thì biến isComplete có thể chuyển thành True rồi.

Chú ý: Trong một số thư viện tính keccak256 có thể khác nhau theo version. Nên ta có thể lấy giá trị bắt đầu của array thông qua transaction set(key=0,value=1) cũng được. Vị trí storage sẽ được lưu trong state của trasaction. 

Bài học kinh nghiệm sau challenge:
1. Vị trí lưu của một array và map ta có thể điều khiển được. Và chúng có thể bị trùng với biến khác vì hàm hash có thể gặp dụng độ.
1. Tất cả đều publish trên blockchain. Ta có thể theo dõi sự thay đổi của contract qua các transaction.




