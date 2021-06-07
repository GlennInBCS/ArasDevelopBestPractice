# Aras Server-side 開發最佳實務 

###### tags: `Aras` `程式開發`

[參考來源: Aras Best Practice](https://community.aras.com/b/english/posts/aras-best-practices-server-side-code)
> Aras開發最重要的兩個原則:
> 
> 1. 向伺服器發出的要求(Request)**越少越好**
> 2. 查詢完後的資料量**越小越好**
> 
> Applying the following best practices to your server-side code improves the consistency and efficiency of your server methods.
> 

## 1. 連同關聯資訊一起搜尋 (Use the Relationship Structure)
![](https://community.aras.com/cfs-file/__key/communityserver-blogs-components-weblogfiles/00-00-00-00-04/rel_2D00_structure.png
)

### 說明:
在查詢物件的時候很常會有需要查詢物件的關聯內容(Relationships)，所以如果已經確定要查詢關聯的話，建議可以在`apply`的時候一起查詢，**盡量不要分開兩次查詢**。

**這邊就用上圖的流程結構為例**

### 🤨 常見的錯誤 (Incorrect)

```C#
// 第一次查詢
Item workflowProcess = this.newItem("Workflow Process", "get");
Item workflowProcessActivity = workflowProcess.createRelationship("Workflow Process Activity", "get");
Item activity = workflowProcessActivity.createRelatedItem("Activity", "get");
activity.setID('activityId');
workflowProcess = workflowProcess.apply();
String workflowProcessId = workflowProcess.getID();

// 第二次查詢
Item workflow = this.newItem("Workflow", "get");
workflowProcess = workflow.createRelatedItem("Workflow Process", "get");
workflowProcess.setID(workflowProcessId);
workflow = workflow.apply();

// ========AML========
// 1st query
// <AML>
//   <Item type="Workflow Process" action="get">
//     <Relationships>
//       <Item type="Workflow Process Activity" action="get">
//         <related_id>
//           <Item type="Activity" action="get" id="activityId"/>
//         </related_id>
//       </Item>
//     </Relationships>
//   </Item>
// </AML>
//
// 2nd query
// <AML>
//   <Item type="Workflow" action="get">
//     <related_id>
//       <Item type="Workflow Process" action="get" id="workflowProcessId" />
//     </related_id>
//   </Item>
// </AML>
```

### 😏 建議作法 ( Best Practice)

👉 **最好的方式就是直接一次查完!!**

```C#
// 一次查詢
Item workflow = this.newItem("Workflow","get");
Item workflowProcess = workflow.createRelatedItem("Workflow Process","get");
Item workflowProcessActivity = workflowProcess.createRelationship("Workflow Process Activity","get");
Item activity = workflowProcessActivity.createRelatedItem("Activity", "get");
activity.setID('activityId');
workflow = workflow.apply();

// ========AML========
// <AML>
//   <Item type="Workflow" action="get">
//     <related_id>
//       <Item type="Workflow Process" action="get">
//         <Relationships>
//           <Item type="Workflow Process Activity" action="get">
//             <related_id>
//               <Item type="Activity" action="get" id="activityId"/>
//             </related_id>
//           </Item>
//         </Relationships>
//       </Item>
//     </related_id>
//   </Item>
// </AML>
```


## 2. 盡量多使用`select` (Use the "Select" Attribute)

### 說明:

我們平常再透過API或是AML查詢Aras的資料的時候，其實很多時候我們都撈了很多**不會**使用到的資料。
其實從Aras送出要求(Request)到Client端收到回覆(Response)的整段時間，撇除http建立連線的時間(其實也還好)，**最耗時的就是從DB拉回來的內容轉換成XML的時間**，所以有目的的減少撈取的資料可以節省不少時間唷~

結論就是，**盡量撈取要使用到的資料就好了**。

### 🤨 常見的錯誤 (Incorrect)
* **當我們沒有下select的時候..**
```xml
<!-- 查詢的結果 -->
<SOAP-ENV:Envelope 
  xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
  <SOAP-ENV:Body>
    <Result>
      <Item type="Workflow" typeId="9E212D4ED3C64493B631EE15D0A62AF7" id="B6A588101AD844C78954452BA44DC4EF">
        <behavior>hard_float</behavior>
        <config_id keyed_name="B6A588101AD844C78954452BA44DC4EF" type="Workflow">B6A588101AD844C78954452BA44DC4EF</config_id>
        <created_by_id keyed_name="Super User" type="User">AD30A6D8D3B642F5A2AFED1A4B02BEFA</created_by_id>
        <created_on>2017-04-04T14:54:06</created_on>
        <generation>1</generation>
        <id keyed_name="B6A588101AD844C78954452BA44DC4EF" type="Workflow">B6A588101AD844C78954452BA44DC4EF</id>
        <is_current>1</is_current>
        <is_released>0</is_released>
        <keyed_name>B6A588101AD844C78954452BA44DC4EF</keyed_name>
        <major_rev>A</major_rev>
        <modified_by_id keyed_name="Super User" type="User">AD30A6D8D3B642F5A2AFED1A4B02BEFA</modified_by_id>
        <modified_on>2017-04-04T14:54:06</modified_on>
        <new_version>1</new_version>
        <not_lockable>0</not_lockable>
        <owned_by_id keyed_name="Mike Miller" type="Identity">E0EFB1EE80724BCDB10E68254ADFF673</owned_by_id>
        <permission_id keyed_name="Express Change - Closed" type="Permission" origPermission="E9340F0B9C3742DD9C75A00EEF2CF3FC">E9340F0B9C3742DD9C75A00EEF2CF3FC</permission_id>
        <related_id keyed_name="DCO-00001001" type="Workflow Process">
          <Item type="Workflow Process" typeId="261EAC08AE9144FC95C49182ACE0D3FE" id="3AAA22CCD01C49C79362E88DB566FD54">
            <active_date>2016-12-01T18:59:00</active_date>
            <closed_date>2016-12-05T15:20:00</closed_date>
            <config_id keyed_name="DCO-00001001" type="Workflow Process">3AAA22CCD01C49C79362E88DB566FD54</config_id>
            <copied_from_string>6CCB9DB9829E4B8993D5881F427854C0</copied_from_string>
            <created_by_id keyed_name="Mike Miller" type="User">E8DA8A7D08BF43B99B55465B5C4AC3BE</created_by_id>
            <created_on>2016-12-01T18:59:00</created_on>
            <css>date-change:created_ondate-change:modified_ondate-change:active_datedate-change:created_ondate-change:modified_ondate-change:closed_date</css>
            <current_state keyed_name="Closed" type="Life Cycle State" name="Closed">46672469D8964E699F24F2AA243FF5B3</current_state>
            <description xml:lang="en">Express DCO Workflow</description>
            <generation>1</generation>
            <id keyed_name="DCO-00001001" type="Workflow Process">3AAA22CCD01C49C79362E88DB566FD54</id>
            <is_current>1</is_current>
            <is_released>0</is_released>
            <keyed_name>DCO-00001001</keyed_name>
            <label xml:lang="en">Express DCO</label>
            <major_rev>A</major_rev>
            <modified_by_id keyed_name="Super User" type="User">AD30A6D8D3B642F5A2AFED1A4B02BEFA</modified_by_id>
            <modified_on>2017-04-04T14:54:32</modified_on>
            <new_version>1</new_version>
            <node_bg_color></node_bg_color>
            <node_label1_color></node_label1_color>
            <node_label1_font/>
            <node_label2_color></node_label2_color>
            <node_label2_font/>
            <node_name_color></node_name_color>
            <node_name_font/>
            <node_size/>
            <not_lockable>0</not_lockable>
            <permission_id keyed_name="Workflow Process Closed" type="Permission" origPermission="C8A6F54842D047F5928C302CFEE26FF9">C8A6F54842D047F5928C302CFEE26FF9</permission_id>
            <process_owner keyed_name="CM" type="Identity">F6624E9AE5504958A84E4B6A5831298B</process_owner>
            <state>Closed</state>
            <transition_line_color></transition_line_color>
            <transition_name_color></transition_name_color>
            <transition_name_font/>
            <name>DCO-00001001</name>
            <Relationships>
              <Item type="Workflow Process Activity" typeId="F3F07BFCCDDF48E79ED239F0111E4710" id="EAD628625E374E64AADD28481313F2A4">
                <behavior>float</behavior>
                <config_id keyed_name="EAD628625E374E64AADD28481313F2A4" type="Workflow Process Activity">EAD628625E374E64AADD28481313F2A4</config_id>
                <created_by_id keyed_name="Super User" type="User">AD30A6D8D3B642F5A2AFED1A4B02BEFA</created_by_id>
                <created_on>2017-04-04T14:54:05</created_on>
                <generation>1</generation>
                <id keyed_name="EAD628625E374E64AADD28481313F2A4" type="Workflow Process Activity">EAD628625E374E64AADD28481313F2A4</id>
                <is_current>1</is_current>
                <is_released>0</is_released>
                <keyed_name>EAD628625E374E64AADD28481313F2A4</keyed_name>
                <major_rev>A</major_rev>
                <modified_by_id keyed_name="Super User" type="User">AD30A6D8D3B642F5A2AFED1A4B02BEFA</modified_by_id>
                <modified_on>2017-04-04T14:54:05</modified_on>
                <new_version>1</new_version>
                <not_lockable>0</not_lockable>
                <permission_id keyed_name="Workflow Process Closed" type="Permission" origPermission="C8A6F54842D047F5928C302CFEE26FF9">C8A6F54842D047F5928C302CFEE26FF9</permission_id>
                <related_id keyed_name="Draft Changes" type="Activity">
                  <Item type="Activity" typeId="937CE47DE2854308BE6FF5AB1CFB19D4" id="E5FB6BE020D247F49927E43BB5788240">
                    <active_date>2016-12-02T13:55:00</active_date>
                    <can_delegate>1</can_delegate>
                    <can_refuse>1</can_refuse>
                    <closed_date>2016-12-05T15:12:00</closed_date>
                    <config_id keyed_name="Draft Changes" type="Activity">E5FB6BE020D247F49927E43BB5788240</config_id>
                    <consolidate_ondelegate>0</consolidate_ondelegate>
                    <created_by_id keyed_name="Super User" type="User">AD30A6D8D3B642F5A2AFED1A4B02BEFA</created_by_id>
                    <created_on>2017-04-04T14:54:05</created_on>
                    <css>date-change:active_datedate-change:modified_ondate-change:closed_date</css>
                    <current_state keyed_name="Closed" type="Life Cycle State" name="Closed">46672469D8964E699F24F2AA243FF5B3</current_state>
                    <generation>1</generation>
                    <icon>../images/WorkflowNode.svg</icon>
                    <id keyed_name="Draft Changes" type="Activity">E5FB6BE020D247F49927E43BB5788240</id>
                    <is_auto>0</is_auto>
                    <is_current>1</is_current>
                    <is_end>0</is_end>
                    <is_escalated>0</is_escalated>
                    <is_released>0</is_released>
                    <is_start>0</is_start>
                    <keyed_name>Draft Changes</keyed_name>
                    <label xml:lang="en">Draft Changes</label>
                    <major_rev>A</major_rev>
                    <managed_by_id keyed_name="Owner" type="Identity">538B300BB2A347F396C436E9EEE1976C</managed_by_id>
                    <message xml:lang="en">Update affected documents</message>
                    <modified_by_id keyed_name="Mike Miller" type="User">E8DA8A7D08BF43B99B55465B5C4AC3BE</modified_by_id>
                    <modified_on>2016-12-05T15:12:00</modified_on>
                    <new_version>1</new_version>
                    <not_lockable>0</not_lockable>
                    <permission_id keyed_name="Workflow Process Closed" type="Permission" origPermission="C8A6F54842D047F5928C302CFEE26FF9">C8A6F54842D047F5928C302CFEE26FF9</permission_id>
                    <state>Closed</state>
                    <wait_for_all_inputs>0</wait_for_all_inputs>
                    <wait_for_all_votes>0</wait_for_all_votes>
                    <x>323</x>
                    <y>55</y>
                    <name>Draft Changes</name>
                  </Item>
                </related_id>
                <sort_order>512</sort_order>
                <source_id keyed_name="DCO-00001001" type="Workflow Process">3AAA22CCD01C49C79362E88DB566FD54</source_id>
              </Item>
            </Relationships>
          </Item>
        </related_id>
        <sort_order>128</sort_order>
        <source_id>38FE441E50824C409282741F0DA9D531</source_id>
        <source_type keyed_name="Express DCO" type="ItemType" name="Express DCO">B678FAF40F7246A7925FB5B116988915</source_type>
        <team_id keyed_name="Product Team" type="Team">862B05F3713F43E793735EDF11D54611</team_id>
      </Item>
    </Result>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```


### 😏 建議作法 ( Best Practice)

**加了`select`的時候，很明顯少了很多內容**
```xml
<!-- result returned by query with select attribute -->
<SOAP-ENV:Envelope 
  xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
  <SOAP-ENV:Body>
    <Result>
      <Item type="Workflow" typeId="9E212D4ED3C64493B631EE15D0A62AF7" id="B6A588101AD844C78954452BA44DC4EF">
        <id keyed_name="B6A588101AD844C78954452BA44DC4EF" type="Workflow">B6A588101AD844C78954452BA44DC4EF</id>
        <source_id>38FE441E50824C409282741F0DA9D531</source_id>
        <source_type keyed_name="Express DCO" type="ItemType" name="Express DCO">B678FAF40F7246A7925FB5B116988915</source_type>
      </Item>
    </Result>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

* **碰到如果是`Item`類型的欄位，特別關聯的`related_id`欄位，常常都會拉一堆Related物件的欄位回來，所以最好也是指定Related Item的欄位** 

```C#
Item workflow = this.newItem("Workflow","get");
// 在Item欄位指定查找的欄位: item_property(property1, property2...)
workflow.setAttribute("select", "related_id(name)");
Item workflowProcess = workflow.createRelatedItem("Workflow Process","get");
Item workflowProcessActivity = workflowProcess.createRelationship("Workflow Process Activity","get");
Item activity = workflowProcessActivity.createRelatedItem("Activity", "get");
activity.setID('activityId');
workflow = workflow.apply();

// generates the following AML query
// same effect as adding select="name" to the Workflow Process item
// <AML>
//   <Item type="Workflow" action="get" select="related_id(name)">
//     <related_id>
//       <Item type="Workflow Process" action="get"> 
//         <Relationships>
//           <Item type="Workflow Process Activity" action="get">
//             <related_id>
//               <Item type="Activity" action="get" id="activityId"/>
//             </related_id>
//           </Item>
//         </Relationships>
//       </Item>
//     </related_id>
//   </Item>
// </AML>

```

* **也可以搭配`maxRecord`指定查詢的筆數，就類似TSQL的`TOP`語法**

```xml
//AML Query
<Item type="Part" action="get" select="item_number, name, unit" maxRecords="100">
</Item>
```


## 3. <a id="idlist"></a>查詢多筆請使用`idlist` (Use the "idlist" Attribute)

### 說明:

我們常常再查詢資料的時候，很多時候都送了多次的要求**一筆一筆**的查詢資料，如果都已經取得所有物件的**ID**就用`idlist`直接一次多筆查詢，就類似TSQL的`in`的語法的概念。


### 🤨 常見的錯誤 (Incorrect)

**請看下面第5行**
```C#
// 上面蒐集完所有ID~~
for (int i = 0; i < openWorkflows.Length; i++)
{
   //取一次ID查詢一次............
   Item openWorkflow = inn.getItemById("workflow process", openWorkflows[i]);
   // 做點事情~
}
// 做更多事情~

// 查詢 #1
// <AML>
//   <Item type="Workflow Process" action="get" id="openWorkflows[0]" />
// </AML>

// 查詢 #2
// <AML>
//   <Item type="Workflow Process" action="get" id="openWorkflows[1]" />
// </AML>

// 查詢好多次了...
// <AML>
//   <Item type="Workflow Process" action="get" id="openWorkflows[n]" />
// </AML>
```


### 😏 建議作法 ( Best Practice)

👉 **最好的辦法就是一次送出去查完!**

```C#
// 上面蒐集完所有ID，組成"逗號"分隔的字串
string inRange = string.Join(",", openWorkflows)));
Item getWorkflows = inn.newItem("workflow process", "get");
// 塞給idlist一次查詢
getWorkflows.setAttribute("idlist", inRange);
getWorkflows = getWorkflows.apply();  
// 做更多事情~

// generates the following AML query, returns one response with all items
// <AML>
//   <Item type="Workflow Process" action="get" idlist="openWorkflows[0],openWorkflows[1],...openWorkflows[n]" />
// </AML>

```


## 4. 盡量避免使用`appendItem()` (Avoid Using appendItem())

### 說明:

以往我們要一次多筆做CRUD(特別是**新增**的時候)的時候就會在`<AML>...</AML>`中間包多筆`Item`，可能就會使用到`appendItem()`的方式來塞Item的AML進去，但...`appendItem(item)`其實本身就會多加一些有的沒的內容，而且在巡覽的方式很慢，很容易讓組出來AML變得很繁雜。

>  it's **slower** and more **cumbersome**.

~~偷偷跟你說...其實我覺得就是原廠這個方法沒寫好，不要說我說的~~

### 🤨 常見的錯誤 (Incorrect)

```C#
Item configuration = inn.newItem("tmp");
for (int i=0; i<10; i++)
{
    Item singleItemConfig = inn.newItem("CAD", "add");
    singleItemConfig.setProperty("item_number", "Test " + i);
    configuration.appendItem(singleItemConfig); //少用
}
configuration.removeItem(configuration.getItemByIndex(0));

Item res = configuration.apply();
```


### 😏 建議作法 ( Best Practice)

**其實最好就是直接組字串送進`appltAML(string)`裡面去，還可以自己控制apped進去的內容，但這邊我跟原廠寫法不同， ~~原廠寫法很爛，~~ 請用我的方式。**

**【原廠】**
```C#
string myAml = "<AML>";
for (int i=0; i<10; i++)
{
    myAml = myAml + "<Item type=\"CAD\" action=\"add\">";
    myAml = myAml + "<item_number>" + "Test " + i + "</item_number>";
    myAml = myAml + "</Item>";
}
myAml = myAml + "</AML>";

Item res = inn.applyAML(myAml);
```

**【Glenn】**

 > 👉 **先設定Template再透過`StringBuilder.AppendLine`的方式組字串**

```C#
string template = @"
    <Item type='CAD' action='add'>
        <item_number>{0}</item_number>
    </Item>
    ";
var aml = new StringBuilder();
aml.AppendLine("<AML>");
for (int i=0; i<10; i++)
{
    aml.AppendLine(string.Format(template, $"Test {i}"));
}
aml.AppendLine("</AML>");

Item res = inn.applyAML(aml);
```


## 5. 多使用`doGetItem="0"` (Use doGetItem="0" with Add and Edit Requests)

### 說明:

當我在Aras的`action`下`add`, `edit`, `update`的時候，執行完後就會把物件的**最終結果**在吐出來一遍。
但...你想想...如果沒有`return error`的話就表示一定成功了或是換句話說...就算有吐回來的內容身為一個~~懶惰的~~優良的工程師，你會再去比對一遍嗎?
:::warning
那乾脆: **沒報錯，只要告訴我你異動是哪一筆，我就當作成功了**
:::

### 🤨 沒有`doGetItem`的時候

**【範例】**
```C#

Innovator inn = this.getInnovator();

Item newPart = inn.newItem("Part", "add");
newPart.setProperty("item_number", "New Part 1");
newPart.setAttribute("doGetItem", "1"); //預設就是1，等於不加
newPart = newPart.apply();

if (newPart.isError())
{
    // 有錯就要處理阿~~~
}
return newPart;

```

**【結果】**
```XML

<Item type="Part" typeId="4F1AC04A2B484F3ABA4E20DB63808A88" id="D5CBA439ED61491F9667840B3A255866">
	<config_id keyed_name="New Part 3" type="Part">D5CBA439ED61491F9667840B3A255866</config_id>
	<created_by_id keyed_name="Innovator Admin" type="User">30B991F927274FA3829655F50C99472E</created_by_id>
	<created_on>2020-12-16T15:57:45</created_on>
	<current_state name="Preliminary" keyed_name="Preliminary" type="Life Cycle State">72A2322564FE4193933CFB5339487A06</current_state>
	<generation>1</generation>
	<has_change_pending>0</has_change_pending>
	<id keyed_name="New Part 3" type="Part">D5CBA439ED61491F9667840B3A255866</id>
	<is_current>1</is_current>
	<is_released>0</is_released>
	<keyed_name>New Part 3</keyed_name>
	<major_rev>A</major_rev>
	<make_buy>Make</make_buy>
	<modified_by_id keyed_name="Innovator Admin" type="User">30B991F927274FA3829655F50C99472E</modified_by_id>
	<modified_on>2020-12-16T15:57:48</modified_on>
	<new_version>0</new_version>
	<not_lockable>0</not_lockable>
	<permission_id keyed_name="New Part" type="Permission" origPermission="5C07EB829D4241F6BB884952960FAF58">5C07EB829D4241F6BB884952960FAF58</permission_id>
	<state>Preliminary</state>
	<unit>EA</unit>
	<item_number>New Part 3</item_number>
	<itemtype>4F1AC04A2B484F3ABA4E20DB63808A88</itemtype>
</Item>

```


### 😏 有`doGetItem`的時候 (建議作法)

**【範例】**
```C#

Innovator inn = this.getInnovator();

Item newPart = inn.newItem("Part", "add");
newPart.setProperty("item_number", "New Part 1");
newPart.setAttribute("doGetItem", "0");
newPart = newPart.apply();

if (newPart.isError())
{
    // 有錯就要處理阿~~~
}
return newPart;

```
**【結果】**
```XML

<Item type="Part" id="3976B16DF29044CFAE02E2915B2AB05B"/>

```

 🤘 **應該可以很明顯看得出差別了吧!**

## 6. 只算數量請用`returnMode="countOnly"` (Use returnMode="countOnly" to Get the Number of Items)

### 說明:

有時候我們查詢不一定是要把所有物件的資料都撈回來，只是想要查一下數量。
例如我們再做**資料分頁**的時候，也就是畫面只秀前10筆但是實際上在資料庫有8萬筆。
所以如果用SQL查總數量就是`select count(*) from XXX`而AML也有類似的查詢語法就是`returnMode="countOnly"`

### 🤨 常見的錯誤 (Incorrect)

在不知道有`returnMode="countOnly"`的情況下，可能是這樣做...
```C#
var inn = this.getInnovator(); 

var parts = inn.newItem("Part", "get");
parts.setProperty("unit", "cm");
parts = parts.apply();
//先全部撈完再算數量...
int count = parts.getItemCount();
```

### 😏 建議作法 ( Best Practice)

**【程式範例】**
```C#
Innovator inn = this.getInnovator();

Item parts = inn.newItem("Part", "get");
parts.setProperty("unit", "cm");
parts.setAttribute("returnMode", "countOnly");
parts.setAttribute("page", "1");
parts.setAttribute("pagesize", "10");
parts = parts.apply();

// 因為returnMode="countOnly"回傳的XML與Item長得不一樣，所以不能直接用getProperty取內容
XmlNode itemMax = parts.dom.SelectSingleNode(".//Message/event[@name='itemmax']/@value");
int itemsCount = int.Parse(itemMax.Value);

return inn.newResult(itemsCount);

//查詢的AML
// <Item type="Part" action="get" returnMode="countOnly" page="1" pagesize="10"/>
//    <unit>cm</unit>
// </Item>
```
**【countOnly查詢的結果】**
```XML
<Message>
  <event name="pagemax" value="1"/>
  <event name="itemmax" value="248"/> <!--我們需要回傳的資料量-->
  <event name="items_with_no_access_count" value="0"/>
</Message>
```


## 7. 避免發多次要求 (Avoid Making Database Calls Within a Loop)

### 說明:

開發最重要的兩個原則的第一點就是盡量少發出Request，所以這邊特別要談的就是不要再迴圈(Loop)中去發要求(CRUD都是)，這個是開發人員很常會忽略的情況。

### 😏 建議作法 ( Best Practice)

請直接參考[第3點](#idlist)



## 8. 最好不要使用`applySQL` (Avoid Using applySQL)

### 說明:

現代很多開發規範其實都禁止工程師直接寫SQL語法...
1. 因為開發人員*不會* 寫SQL語法 
2. SQL沒寫好很容易就造成效能問題
3. 程式碼裡面參雜SQL語法不好維護 
4. SQL容易造成資安問題 (**SQL Injection**)

那在Aras裡面呢?

1. SQL會直接繞過Aras中設定的權限規則
2. 前端程式碼(Javascript)不能直接下SQL語法

### 🤨 盡量不要這樣寫

```C#
Innovator inn = this.getInnovator();
string sql = "SELECT * FROM innovator.[PART] WHERE item_number='PART-00001'";
Item part = inn.applySQL(sql);
return part;

```

### 😏 建議作法 (Best Practice)


* 👍 **盡量使用Aras的API/AML來進行CRUD**

```C#
Item part = this.newItem("Part","get");
part.setProperty("item_number","PART-00001");
part = part.apply();
return part;

// 或者是這樣寫...

Innovator inn = this.getInnovator();
string aml = "<AML><Item type='Part' action='get'><item_number>PART-00001</item_number></Item></AML>";
Item part = inn.applyAML(aml);
return part;
```

* 👉 **但是畢竟AML並不像SQL語法這麼彈性，有些情況還是無法達到，那會建議就包成`Store Procedure`來呼叫，達到隔離程式碼與SQL語法的目的**

```C#
Item sql = this.newItem("SQL","SQL PROCESS");
sql.setProperty("name","My SQL Procedure Item");
sql.setProperty("PROCESS","CALL");
Item sql_result = sql.apply();
return sql_result;
```
* 👉 **使用12SP8版以後新的方法: `applySQLWithParameters`**

⛔⛔ <b style="color:red;">【注意】此方法是Aras 12.0 SP8以後才有的，11.0是沒有的喔!!</b>


很多企業會要求開發人員在寫SQL語法的時候，`where`條件不可以直接組合前方條件。
```C#
using(var conn = new SqlConnection("Server=.;Database=XXX;User Id=aaa;Password=123"));
{
      conn.Open();
      //這樣完全不合格，肯定會造成SQL Injection!
      string sql = "select * from innovator.Part where id=''" + form.getID() + "'  and created_on > '" + today + "'";
      var cmd = new SqlCommand(sql, conn);
      var reader = cmd.ExcuteReader();
      // 讀資料做事...
}
```
通常會要求一定要用`SqlParameter`[^sqlparam]來指派`where`條件內容
```C#
using(var conn = new SqlConnection("Server=.;Database=XXX;User Id=aaa;Password=123"))
{
      conn.Open();
      //這樣完全不合格，肯定會造成SQL Injection!
      string sql = "select * from innovator.Part where id=@id and created_on > @today";
      var cmd = new SqlCommand(sql, conn);
      //設定where條件的內容
      cmd.Parameters.AddWithValue("@id", form.getID());
      cmd.Parameters.AddWithValue("@today", today);
      var reader = cmd.ExcuteReader();
      // 讀資料做事...
}
```
12版終於有這個方法啦~👏

```C#

var inn = this.getInnovator();
string sql = "select name, id, major_rev from innovator.ITEMTYPE where name = @searchName";

string param = "<Parameters><Parameter name='searchName' type='string'>Access</Parameter></Parameters>";

//<Parameters>
//    name就是你設定的參數名稱； type就是Aras的DataType; Parameter就是值
//    <Parameter name='name' type='string'>Access</Parameter>
//</Parameters>

return inn.applySQLWithParameters(sql, param);
```
[^sqlparam]: https://docs.microsoft.com/zh-tw/dotnet/api/system.data.sqlclient.sqlparameter?view=dotnet-plat-ext-5.0
---

# 提高維護性(Maintainability

## 1. 盡量使用Innovator API (Use the Innovator API)

### 說明:

在Aras開發一定會用到IOM裡面的方法`newItem()`,`apply()`,`getPorpeorty`...

🤔❔ **想想看...有沒有甚麼非不用的理由?或是說好處是甚麼呢?**

1. Aras已經公開這個介面了，基本上是不太可能會改變，至少方法名稱是如此
2. 不會因為Aras版本升級或是換了底層內容而開發人員需要把程式碼翻掉重改
3. 易讀性比較高，只要是Aras的開發人員一定看得懂
4. 比起自己動手組AML字串容易許多，不用因為`'(單引號)`, `"(雙引號)`傷透腦筋
5. 方法可以偵錯(Debuggable)
6. 不用直接寫SQL語法，也避免掉SQL的資安問題

> 不過...有時候Aras沒有在IOM提供API，而開發人員可能會直接去使用其他元件，比如說**Aras.Server.Core**裡面的方法，那就有心理準備未來可能會需要修改程式碼囉~


## 2. 可以多使用快捷方法 (Use Shorthand Functions)

### 說明:

* 甚麼是Shorthand Function?
* 為什麼要多用Shorthands?

### 😏 Attribute Shorthand Functions

```C#
Innovator inn = this.getInnovator();
Item part = inn.newItem();

/* 下面兩兩一組是相同的意思 */
// Set the type attribute
part.setAttribute("type", "Part");
part.setType("Part");

// Set the action attribute
part.setAttribute("action", "get");
part.setAction("get");

// Set the ID attribute
part.setAttribute("id", "00995D4043BC48CE9D869D78C17CF0D6");
part.setID("00995D4043BC48CE9D869D78C17CF0D6");

/* Similarly, there are corresponding getType, getAction, and getID functions */
string type, action, id;

// Get the type attribute
type = part.getAttribute("type");
type = part.getType();

// Get the action attribute
action = part.getAttribute("action");
action = part.getAction();

// Get the ID attribute
id = part.getAttribute("id");
id = part.getID();
```


### 😏 Items Shorthand Functions


```C#
Innovator inn = this.getInnovator();
Item part = inn.newItem("Part", "get");

// 建立Part BOM關聯查詢
Item partBom = inn.newItem("Part BOM", "get");
part.addRelationship(partBom);
// 快捷方法
Item partBom = part.createRelationship("Part BOM", "get");

// 建立Item屬性的查詢
Item createdBy = inn.newItem("User", "get");
part.setPropertyItem("created_by_id", createdBy);
// 快捷方法
Item createdBy = part.createPropertyItem("created_by_id", "User", "get");

// 建立related物件的查詢
Item relatedPart = partBom.createPropertyItem("related_id", "Part", "get");
// 快捷方法
Item relatedPart = partBom.createRelatedItem("Part", "get");
```



## 3. 盡量不要在程式碼寫死固定值 (Avoid Hardcoding)

### 🔟 魔術數字(Magic Number):

在 _開發大全([Complete Code]_(https://en.wikipedia.org/wiki/Code_Complete))裡有提到關於**魔術數字** [^magicno] 的議題:
>Avoid "magic numbers" Magic numbers are literal numbers, such as 100 or 47524, that appear in the middle of a program without explanation. If you program in a language that supports named constants, use them instead. If you can't use named constants, use global variables when it's feasible to do so.
>
>Avoiding magic numbers yields three advantages:
> * Changes can be made more reliably. If you use named constants, you won't over-look one of the 100s or change a 100 that refers to something else.
> * Changes can be made more easily. When the maximum number of entries changes from 100 to 200, if you're using magic numbers you have to find all the 100s and change them to 200s. If you use 100+1 or 100-1, you'll also have to find all the 101s and 99s and change them to 201s and 199s. If you're using a named constant, you simply change the definition of the constant from 100 to 200 in one place.
> * Your code is more readable. Sure, in the expression
> 
> _Code Complete Chapter 2_

**直接來看看範例吧!**

```C#
//請問這段程式碼代表甚麼意思?
double total = 1000 * 0.05;
```

**如果程式碼是這樣呢?**

```C#
// 應該大概可以猜得出來再算"總金額"
const double TAX = 0.05;
double total = 1000 * TAX;
```

👉 簡單說就是用讓**其他人**可以**看得懂**的方式來呈現，如: `double circleArea = r * Math.PI;`


[^magicno]: https://en.wikipedia.org/wiki/Magic_number_(programming)

### 🆖 寫死固定值

```C#
//我們通常會這樣寫Code
Item part = inn.newItem("Part", "get");
part.setID(this.getID());
part.setAttribute("select", "locked_by_id(keyed_name)");
part = part.apply();
```
**如果這段程式碼是很多Itemtype都是相同邏輯的話...是不是可以改成通用呢?**
```C#
//把取Type改成動態取，不要寫死
string thisType= this.getType();
Item part = inn.newItem(thisType, "get");
part.setID(this.getID());
part.setAttribute("select", "locked_by_id(keyed_name)");
part = part.apply();
```

---
# 額外的建議

## 1. 不要使用ActiveX控件 

### 說明:

這個建議是從Aras原廠再另外一篇2017的[文章](https://community.aras.com/b/english/posts/aras-best-practices-community-projects-part-2)，是寫給要貢獻Community Project的開發人員的建議。
不過各位其實不用太擔心會用到ActiveX，因為ActiveX只存在於**IE** [^active_ie]，其實正確來說只是大部分被用在IE，但其實ActiveX是COM元件[^com]，他的概念就像是那個~~討厭的~~Adobe Flash Player一樣，裝一個程式執行在瀏覽器背後。
**但是最主要是使用舊版的客戶(9.X, 10.X, 11.X)，特別是9版的客戶可常會用到，現在就是不能用囉~**



[^active_ie]: https://support.microsoft.com/en-us/windows/use-activex-controls-for-internet-explorer-11-25738d05-d357-39b4-eb2f-fdd074bbf347
[^com]: https://en.wikipedia.org/wiki/Component_Object_Model

## 2. 抓取遞迴式結構

### 說明:

在Aras系統中**多階BOM結構**，**多階CAD結構**或是**組織結構**都是屬於遞迴式結構(_自己在找自己_)，身為開發工程師的你會怎麼抓這樣的結構呢?
1. 寫遞迴方法
1. 寫CTE的SQL語法
2. ...

其實Aras已經幫妳寫好了~😏 

### 請看下方AML
在Source物件使用`action='GetItemRepeatConfig'`，並在Relationships的關聯物件加入`repeatProp='related_id'`及` repeatTimes='{遞迴的次數}'`
```xml
<AML>
	<Item type='Part' select='item_number,cn_tag' action='GetItemRepeatConfig' id='ED93E6877FEC49F2BF5F8C750A0B6E71'>
		<Relationships>
			<Item type='Part BOM' select='related_id,quantity' repeatProp='related_id' repeatTimes='10'/>
		</Relationships>
	</Item>
</AML>
```


## 3. 建議使用XDocument 

### 說明:


**這個存粹是我個人建議，要服務此藥方請確認可以善後! 
~~開發一定有風險，使用技術有好有壞，寫Code前應詳閱API說明書~~** 😉

* Aras背後是使用SOAP的機制，SOAP背後就是XML的內容，所以Aras所有的元件都是透過 .NET Framework的`System.Xml.XmlDocument`來處理XML的內容(IOM中的`setProperty`、`setAttribute`也就是在控制XML)。
* 而`XmlDocument`已經是舊式的API (.NET 2)，所以在處理大篇幅的XML內容會相對比較慢，所以在處裡上比較沒有*批量* 的處理方式(大多是迴圈一筆一筆讀)，而且非常依賴**XPath**[^xpath]的指令。
* 所以...如果我在寫外部的元件或是執行檔的話，我會習慣把XML的內容轉成`XDocument`的物件(.NET 4)來如處裡。[^xdocument]

> 那使用`XDocument`的**好處**是甚麼?
>  1. 處理上速度相對比較快
>  2. 寫起來比較精簡，可讀性比較高
>  3. 可以搭配LINQ一起使用 👍
   
我把`applySQLWithParameter`的範例改用`XmlDocument`來看看
```C#
var inn = this.getInnovator();
string sql = "select name, id, major_rev from innovator.ITEMTYPE where name = @name";

var _params = inn.newXMLDocument(); //也是可以用new XmlDocument()啦
var root = _params.CreateElement("Parameters");
_params.AppendChild(root);

//可能是跑回圈設定一堆輸入參數...
var param = _params.CreateElement("Parameter");
param.SetAttribute("name", "name");
param.SetAttribute("type", "string");
param.InnerText = "Access";
root.AppendChild(param);

return inn.applySQLWithParameters(sql, _params.InnerXml);
```
如果是使用`XDocument`呢?
```C#
var inn = this.getInnovator();
string sql = "select name, id, major_rev from innovator.ITEMTYPE where name = @name";
//必須要引用System.Xml.Linq
var _params = new XDocument("Parameters")


//可能是跑回圈設定一堆輸入參數...
_params.Root.Add(new XElement("Parameter", 
                              new XAttribute("name", "name"),
                              new XAttribute("type", "string"),
                              "Access"));

return inn.applySQLWithParameters(sql, _params.ToString());
```
   
[^xdocument]: https://docs.microsoft.com/en-us/dotnet/standard/linq/linq-xml-vs-dom
[^xpath]: https://www.w3school.com.cn/xpath/xpath_syntax.asp
