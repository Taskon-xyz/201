**邀请Dashboard**

**需求背景**

有一些给我们导流的合作方，按照交易量给返佣，需要设计一整套流程，来满足合作方的需求。

**基本流程**

创建邀请链接

可自动跳转到活动页面（由配置决定）

合作方分发邀请链接

用户通过邀请链接访问，直接跳转到活动页面

用户注册，记录邀请关系

用户交易，根据邀请关系，有不同的手续费费率设置

合作方查看邀请用户数据和交易量

手动结算返佣金额

**需求详情**

1\. **邀请Dashboard页**

<img src="./media/media/image1.png" style="width:5.75in;height:3.inspire-shot" />

***图片描述***:
该图片展示了“邀请Dashboard”的界面原型。主要包含以下几个部分：

- **顶部**: 显示合作伙伴的邀请链接（Invite Link）和邀请码（Invite Code）。
- **数据概览区**: 以卡片形式展示了关键指标，包括：
  - **Register Users**: 注册用户总数。
  - **Trading Users**: 产生过交易的用户数。
  - **Trading Volume**: 邀请用户的总交易量。
  - **Trading Fee**: 邀请用户产生的总交易费用。
- **筛选与列表区**:
  - 提供了时间筛选功能（Yesterday, 7D, 30D）。
  - 下方是一个“Detail Data”表格，用于展示详细的交易记录，包含时间（Time）、网络（Network）、交易金额（Trading
    Value）、交易费用（Trading Fee）和用户地址（Address）等字段。
  - 右上角有一个“Download”按钮，用于导出数据。

---

Invite Link邀请链接

是一个自带重定向跳转的邀请链接，而且重定向之后邀请code不会丢失

默认跳转到chainearn页面

可以通过运营工具配置跳转到其他页面，比如某个活动的页面

Register Users：通过该邀请码注册的用户数

Trading Users：通过该邀请码注册的用户中，在taskon交易过的用户数

Trading Volume：邀请的用户，交易量总和

Trading Fee：邀请的用户，产生的交易费

不同的邀请码，带来的用户，可以设置不同的交易费比例，通过配置文件设置就可以了

【同一个邀请码，不同的活动，是否可以设置不同的交易费比例？】

这里展示的Trading Fee，是扣除了给OO的分成，只展示属于Taskon的部分

时间筛选区域

下拉框里面是Yesterday、7D、30D，默认选中Yesterday

选择Yesterday的数据，展示过去一个自然日的数据，按照UTC+0计算

选择7D，则展示最近7D内的数据

选择30D，则展示最近30D的数据

Detail Data，展示邀请的用户，所有的交易明细，每页50条，展示所有记录

Time：交易时间 ~~【是否加上注册时间】~~

Network：所在链

Trading Value：交易金额，按USD计算

Trading Fee：产生的交易费

Address：用户地址

Download，点击把交易记录下载为CSV，字段格式一样

该网页可以先采用拼接URL的模式展示，不提供C端入口，URL格式：taskon.xyz/invitedashboard/ + xxxxinvitecode
