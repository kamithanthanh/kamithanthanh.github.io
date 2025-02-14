---
layout : post
title : Fuzzing simple program with angr 
--- 

# Mở đầu   
Chúc các đạo hữu năm mới khoái hoạt. 😁😁😁 
Khai xuân đầu năm mình có học một chút về [```angr```](https://docs.angr.io/). Cái này cũng đọc lâu rồi nhưng xưa thấy khó quá với lười nên lại thôi. Đầu năm tinh thần phấn khởi, lại tìm được mấy bài [writeup](https://blog.efiens.com/tag/ctf/) về angr của team efiens thấy khá hay và dễ hiểu nên mình quyết định tìm hiểu thêm về nó.🥳🥳🥳   
Sau khi đọc xong writeup trên thì mình có tổng kết sơ lại được các bước thực hiện của Angr tại [đây](https://www.notion.so/Basic-Setup-998957b22a5a4c05a077a4851b2e1da0)   


# Simple Program   
Quay trở lại đề tài, sau khi nắm bắt được một số bước cơ bản tiếp cận angr, mình lại tiếp tục cày [tutorial này](https://github.com/jakespringer/angr_ctf/tree/master/solutions). Nó cho ta những challenge dạng ctf đơn giản và nhưng công cụ thực hiện khác nhau giúp ta nắm bắt thêm các cách sử dụng linh hoạt của angr trong từng trường hợp khác nhau.🙂🙂🙂 Cuối cùng,ở challenge 17 nó có trình bày một bài fuzzing và mình thấy khá là thú vị nên note lại ở đây.   
Chương trình bao gồm 2 hàm cơ bản :   

![](/ctf/temp/fuzzAngr1%20(1).PNG)    

![](/ctf/temp/fuzzAngr1%20(2).PNG)

Mục tiêu của chương trình này là khai thác lỗi để in ra :   

![](/ctf/temp/fuzzAngr1%20(3).PNG)    

Chương trình có một lỗi overflow cơ bản. Nhưng nó rất hợp để làm ví dụ mở đầu.  
Giả sử chưa biết lỗi overflow, mà dựa trên yêu cầu chúng ta biết được bằng cách nào đó chúng ta phải tìm được cách thay đổi luồng thực thi của chương trình để nó gọi hàm ```print_good```.   

## Under-constrained state   

Trong khi chương trình được thực hiện bởi angr, Under-constrained state xảy ra khi thanh ghi EIP mang giá trị tượng trưng (có nghĩa là bị ảnh hưởng bởi user-input). Đây là những trạng thái chúng ta cần quan tâm trong trường hợp này.  
Để kiểm tra, chúng ta có thể dùng đoạn code sau :   
```python
 def check_vulnerable(state):
    return state.se.symbolic(state.regs.eip)
```

## Stash  

Stash là một danh sách phân loại các trạng thái.Bao gồm :  

  + 'active' : trạng thái mà chương trình có thể tiếp tục thực hiện
  + 'deadended' : trạng thái kết thúc chương trình
  + 'errored' : trạng thái chương trình gặp lỗi với angr
  + 'unconstrained' : Under-constrained state
  + 'unsat' : trạng thái không thể tồn tại (nghĩa là phương trình vô nghiệm)  

Chúng ta có thể tự cấu hình stash như sau :  

```python
simulation = project.factory.simgr(
    initial_state, 
    save_unconstrained=True,
    stashes={
      'active' : [initial_state],
      'unconstrained' : [],
      'found' : [],
      'not_needed' : []
    }
  )
```  

Tất cả các loại stash không cần thiết chúng ta cho hết vào danh sách ```not_need```.  
Mặc định, Angr sẽ hủy bỏ những trạng thái unconstrained. Chúng ta có thể điều chỉnh bằng cách ```save_unconstrained=True```. Khi đó, Angr sẽ lưu các trạng thái đó vào ```simulation.unconstrained```.   

## Fuzzing Step

+ Giai đoạn 1 : Thu thập tất cả các unconstrained state:  
  - Thực hiện dịch chuyển tất cả stash ```unconstrained``` sang stash ```found```  

  ```c
  simulation.move('unconstrained', 'found')
  ```  

  - Tiếp tục thực hiện chương trình bằng lệnh ```simulation.step()```
  - Lặp cho tới khi không còn trạng thái active hoặc trạng thái unconstrained thì dừng.  
Việc ```step``` hoạt động như nào lại là vấn đề sâu xa mà mình cũng chưa rõ cách hoạt động của nó 😥 Nói chung, Angr sẽ thực hiện chọn input đầu vào là các biến, thực hiện chương trình là các biến đó và theo dõi quá trình hoạt động từ đầu đến cuối. Ở mỗi bước thực hiện, tiến hành lọc ra tất cả các trạng thái uncóntrained thu được và lưu tại stash ```found``` để sau đó có thể truy cập được thông qua ```simulation.found```.  
+ Giai đoạn 2 : Với một trạng thái ```found``` tìm được, chúng ta tiến hành thêm ràng buộc của eip phải trỏ tới hàm ```print_good```.  

  ```python
    solution_state = simulation.found[0]
    solution_state.add_constraints(solution_state.regs.eip == print_good_addr)
  ```
Cuối cùng là in ra kết quả tìm được :  
  ```python
    solution = solution_state.posix.dumps(sys.stdin.fileno())
    print(solution)
  ```

# Kết thúc   
😁😁😁 Vậy đó là một vài bước căn bản để fuzz được một chương trình cơ bản bằng angr. Mình sẽ tiếp tục tìm hiểu thêm về nó😁😁😁 Phần khó khăn vẫn đang chờ ở phía sau :)))   


