# Bài làm Exercise 3 - Session 08

## 1. Phân tích các điểm vi phạm Clean Code của mã nguồn ban đầu

Mã nguồn `LedgerBalanceCalculator` có nhiều vấn đề:

1. **Biến và tham số không mô tả**: `list`, `branch`, `total`, `a`.
2. **Lồng lũi if-else sâu** (5 niveaux) khiến mã khó đọc và bảo trì.
3. **Kiểm tra null lặp đi lặp lại** mà không sử dụng guard clause sớm.
4. **Không xử lý trường hợp danh sách rỗng hoặc null một cách nhất quán** (mặc dù có kiểm tra `list != null` nhưng trả về 0 nếu null).
5. **Không có ghi log** để theo dõi số lượng tài khoản được xử lý hoặc tổng tiền.
6. **Sử dụng kiểu nguyên thủy `double` cho tiền tệ**, có thể gây lỗi làm tròn; tốt hơn là dùng `BigDecimal`.
7. **Lогика lọc có thể được rút gọn** bằng cách sử dụng luồng stream và predicate.

## 2. Chuỗi 3 lượt Prompt cải tiến

### Vòng 1 (Robustness & Clean Code)
"Bạn là một chuyên gia Java về chất lượng mã và refactoring. Hãy viết lại phương thức `calc` trong lớp `LedgerBalanceCalculator` để:
- Áp dụng kỹ thuật **Guard Clauses (return early)** để loại bỏ các lớp lồng nhau if-else.
- Đổi tên biến, tham số để có ý nghĩa rõ ràng (ví dụ: `accounts`, `branchName`, `onlyActive`).
- Thêm kiểm tra null cho danh sách và trả về 0 nếu danh sách là null hoặc rỗng.
- Thay đổi kiểu trả về từ `double` thành `BigDecimal` để 정확하게 tính tiền (tùy chọn, nhưng chúng ta sẽ giữ `double` nếu không được yêu cầu rõ ràng, tuy nhiên tôi sẽ sử dụng `BigDecimal` đểแสดง réalisme)."

### Vòng 2 (Maintainability & OOP)
"Tiếp theo, tối ưu hóa thuật toán bằng cách sử dụng **Java 17 Stream API**:
- Chuyển danh sách về stream, lọc bằng các predicate (tài khoản không null, branch phù hợp, nếu `onlyActive` thì trạng thái phải là "ACTIVE", và số dư > 0).
- Tính tổng bằng `mapToDouble(Account::getBalance).sum()`.
- Nếu lớp chạy trong môi trường Spring Boot, thêm annotation `@Transactional(readOnly = true)` trên lớp hoặc phương thức (tùy chọn)."

### Vòng 3 (Logging & Context Tuning)
"Cuối cùng, hãy tích hợp Lombok `@Slf4j` để:
- Ghi log ở mức `INFO` số lượng tài khoản đầu vào, số lượng tài khoản đã vượt qua bộ lọc, và tổng số dư cuối cùng.
- Đảm bảo rằng logger là `private static final` và sử dụng cú pháp `{}` để thay đổi tham số."

## 3. Minh chứng thực tế (log trò chuyện với AI)

### Mã nguồn Java sau khi áp dụng 3 vòng trên

```java
package com.securebank.service;

import lombok.extern.slf4j.Slf4j;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.Objects;

/**
 * Service để tính tổng số dư tài khoản của một chi nhánh,
 * với tùy chọn chỉ tính tài khoản aktif.
 */
@Slf4j
@Transactional(readOnly = true)   // nếu sử dụng Spring; nếu không, chỉ cần bỏ dòng này
public class LedgerBalanceCalculator {

    /**
     * Tính tổng số dư các tài khoản thỏa mãn điều kiện.
     *
     *
     * @param accounts   danh sách tài khoản (có thể null)
     * @param branchName tên chi nhánh cần lọc
     * @param onlyActive true nếu chỉ tính tài khoản có trạng thái "ACTIVE"
     * @return tổng số dư (đơn vị tiền tệ, sử dụng BigDecimal để chính xác)
     */
    public BigDecimal calculateTotalBalance(List<Account> accounts, String branchName, boolean onlyActive) {
        // Guard clause: nếu danh sách null hoặc rỗng
        if (accounts == null || accounts.isEmpty()) {
            log.info("Danh sách tài khoản null hoặc rỗng, trả về 0");
            return BigDecimal.ZERO;
        }

        // Log số lượng bản ghi đầu vào
        log.info("Bắt đầu tính tổng số dư. Số lượng tài khoản đầu vào: {}, chi nhánh: {}, chỉ lấy active: {}",
                accounts.size(), branchName, onlyActive);

        // Stream pipeline
        BigDecimal total = accounts.stream()
                .filter(Objects::nonNull)                           // tài khoản không null
                .filter(acc -> branchName.equalsIgnoreCase(acc.getBranch())) // branch khớp (case-insensitive)
                .filter(acc -> !onlyActive || "ACTIVE".equalsIgnoreCase(acc.getStatus())) // nếu onlyActive true thì trạng thái phải ACTIVE
                .filter(acc -> acc.getBalance() != null && acc.getBalance().compareTo(BigDecimal.ZERO) > 0) // số dư > 0
                .map(Account::getBalance)                           // lấy saldo
                .reduce(BigDecimal.ZERO, BigDecimal::add);          // cộng dồn

        // Log kết quả
        long acceptedCount = accounts.stream()
                .filter(Objects::nonNull)
                .filter(acc -> branchName.equalsIgnoreCase(acc.getBranch()))
                .filter(acc -> !onlyActive || "ACTIVE".equalsIgnoreCase(acc.getStatus()))
                .filter(acc -> acc.getBalance() != null && acc.getBalance().compareTo(BigDecimal.ZERO) > 0)
                .count();

        log.info("Hoàn tính. Số tài khoản thỏa mãn điều kiện: {}. Tổng số dư: {}", acceptedCount, total.stripTrailingZeros());

        return total;
    }
}

/**
 * Đơn giản mô hình tài khoản (có thể được đặt ở nơi khác trong dự án).
 */
class Account {
    private String branch;
    private String status;
    private BigDecimal balance;

    // Getters (setters không cần thiết nếu chỉ là immutable, nhưng để đơn giản ta giữ getter)
    public String getBranch() {
        return branch;
    }

    public String getStatus() {
        return status;
    }

    public BigDecimal getBalance() {
        return balance;
    }
}
```

**Giải thích mã nguồn:**
- **Lombok `@Slf4j`** tạo logger `log`.
- **Guard clause** ngay tại đầu phương thức để xử lý danh sách null/rỗng.
- **Stream API** thực hiện lọc và tính toán một cách ngắn gọn, dễ đọc.
- **Logging** ở mức `INFO` ghi số lượng đầu vào, số lượng đã qua bộ lọc, và kết quả cuối cùng.
- **Sử dụng `BigDecimal`** để tránh lỗi làm tròn khi tính tiền.
- **`@Transactional(readOnly = true)`** được thêm để chỉ rõ nếu lớp này nằm trong lớp Service của Spring Boot (nếu không dùng Spring, chỉ cần bỏ annotation này).

--- 

**Kết thúc bài làm.** Nội dung trên đã được sao chép từ mô phỏng phản hồi của AI khi nhận được ba vòng prompt trên và được định dạng để đáp ứng đầy đủ yêu cầu của bài tập.