################################################
### IBancorConverter 对象

关联对象

IERC20Token;

IWhitelist;


	/**
	@dev returns the expected return for converting a specific amount of _fromToken to _toToken（返回将特定金额从令牌转换为IO到令牌的预期回报）
	
        @param _fromToken  ERC20 token to convert from
        @param _toToken    ERC20 token to convert to
        @param _amount     amount to convert, in fromToken（从FoT中转换的金额）
        @return expected conversion return amount（返回预期转换收益金额）
	*/
	function getReturn(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount) public view returns (uint256);

	/**
        @dev converts a specific amount of _fromToken to _toToken（将一个特定量的从）
        @param _fromToken  ERC20 token to convert from
        @param _toToken    ERC20 token to convert to
        @param _amount     amount to convert, in fromToken  （从FoT中转换的金额）
        @param _minReturn  if the conversion results in an amount smaller than the minimum return - it is cancelled, must be nonzero（如果转换结果小于最小返回量，则被取消，必须为非零。）
        @return conversion return amount（返回转换返回量）
	*/
	function convert(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn) public returns (uint256);


	function conversionWhitelist() public view returns (IWhitelist) {}
	function conversionFee() public view returns (uint32) {}
	function connectors(address _address) public view returns (uint256, uint32, bool, bool, bool) {}
	function getConnectorBalance(IERC20Token _connectorToken) public view returns (uint256);
	function change(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn) public returns (uint256);

#################################################
### IBancorConverterFactory 对象
关联对象

	IERC20Token;

	ISmartToken;

	IContractRegistry；

	function createConverter(
        ISmartToken _token,
        IContractRegistry _registry,
        uint32 _maxConversionFee,
        IERC20Token _connectorToken,
        uint32 _connectorWeight
    	)
	public returns (address);
###################################################
### IBancorFormula  对象

	function calculatePurchaseReturn(uint256 _supply, uint256 _connectorBalance, uint32 _connectorWeight, uint256 _depositAmount) public view returns (uint256);
	function calculateSaleReturn(uint256 _supply, uint256 _connectorBalance, uint32 _connectorWeight, uint256 _sellAmount) public view returns (uint256);
	function calculateCrossConnectorReturn(uint256 _fromConnectorBalance, uint32 _fromConnectorWeight, uint256 _toConnectorBalance, uint32 _toConnectorWeight, uint256 _amount) public view returns (uint256);
#########################################
### IBancorGasPriceLimit  对象

	function gasPrice() public view returns (uint256) {}
	function validateGasPrice(uint256) public view;


### IBancorConverter (源码)
#####################################################

BANCOR变换器V0.10

	令牌转换器的BANCOR版本允许在智能令牌和其他Erc20令牌之间以及在不同的Erc20令牌和它们自己之间进行转换。

	ErC20连接器的平衡可以是虚拟的，这意味着计算是基于虚拟平衡而不是依靠。

	实际连接器平衡。这是一种安全机制，可以防止在单个合同中保持非常大的（和有价值的）平衡。

	转换器是可升级的（就像任何StasktoKeNo控制器）一样。

	警告：不建议使用具有小于8小数位数的智能令牌的转换器。

	或由于精度损失而具有非常小的数目

	开放性问题：

	前面运行的攻击目前被下列机制减轻：

	每个转换的最小返回参数提供了一种定义事务的最小/最大价格的方法。

	气体价格限制防止用户对执行命令的控制

	如果交易来自可信的白名单签名者，则可以跳过气体价格限制检查。

	其他潜在的解决方案可能包括基于提交/披露的方案。

	-可能为连接器字段添加吸气剂，这样客户端就不必依赖于结构中的顺序。

### BancorConverter源码分析


BancorConverter 继承:IBancorConverter, SmartTokenController, Managed, ContractIds, FeatureIds属性


	uint32 private constant MAX_WEIGHT = 1000000;

	uint64 private constant MAX_CONVERSION_FEE = 1000000;

	struct Connector {

        uint256 virtualBalance;         // connector virtual balance（资产表的链接）
	
        uint32 weight;                  // connector weight, represented in ppm, 1-1000000（连接器权值，以PPM表示）
	
        bool isVirtualBalanceEnabled;   // true if virtual balance is enabled, false if not（如果可以启动是true，如果不是则为false）
	
        bool isPurchaseEnabled;         // is purchase of the smart token enabled with the connector, can be set by the owner（可以由所有者设置，是否开始智能令牌的购买）
	
        bool isSet;                     // used to tell if the mapping element is defined（判断交易是否继续）
	}

	string public version = '0.10';

	string public converterType = 'bancor';


 	IContractRegistry public registry;                  // contract registry contract（合同相互登记）
 
 	IWhitelist public conversionWhitelist;              // whitelist contract with list of addresses that are allowed to use the 		converter（与允许使用转换器的地址列表的白名单契约）
 
 	IERC20Token[] public connectorTokens;               // ERC20 standard token addresses（Erc20标准令牌地址 ）
 
 	IERC20Token[] public quickBuyPath;                  // conversion path that's used in order to buy the token with ETH（使用ETH购买令牌的转换路径）
 
	 mapping (address => Connector) public connectors;   // connector token addresses -> connector data（连接器令牌地址>连接器数据）
 
	 uint32 private totalConnectorWeight = 0;            // used to efficiently prevent increasing the total connector weight above 100%（用于有效防止总连接器重量增加到100%以上）
 
	 uint32 public maxConversionFee = 0;                 // maximum conversion fee for the lifetime of the contract,
                                                        // represented in ppm, 0...1000000 (0 = no fee, 100 = 0.01%, 1000000 = 100%)（合同期限内的最大转换费用，以PPM表示，0…1000000（0＝无费用，100＝0.01%，1000000＝100%）。 ）
							
 	uint32 public conversionFee = 0;                    // current conversion fee, represented in ppm, 0...maxConversionFee（当前转换费，以PPM表示，0……最大转换费 ）
 
	bool public conversionsEnabled = true; 			// true if token conversions is enabled, false if not（如果启用令牌转换，则为false，如果不是）



	IERC20Token[] private convertPath;

   


    // triggered when a conversion between two tokens occurs(Token转换时候的触发事件)
    event Conversion(
        address indexed _fromToken,
        address indexed _toToken,
        address indexed _trader,
        uint256 _amount,
        uint256 _return,
        int256 _conversionFee
	);

    // triggered after a conversion with new price data（触发后获取转换后新的价格 ）
    event PriceDataUpdate(
        address indexed _connectorToken,
        uint256 _tokenSupply,
        uint256 _connectorBalance,
        uint32 _connectorWeight
	);

    // triggered when the conversion fee is updated (费用更新时候触发事件)
	event ConversionFeeUpdate(uint32 _prevFee, uint32 _newFee);



    /**
        @dev constructor（构造函数）
        @param  _token              smart token governed by the converter（由转换器控制的智能令牌）
        @param  _registry           address of a contract registry contract（合同地址）
        @param  _maxConversionFee   maximum conversion fee, represented in ppm（最大转换费，以PPM表示 ）
        @param  _connectorToken     optional, initial connector, allows defining the first connector at deployment time（可选的初始连接器，允许在部署时定义第一个连接器。）
        @param  _connectorWeight    optional, weight for the initial connector（可选的，初始连接器的重量）
    **/
    constructor(
        ISmartToken _token,
        IContractRegistry _registry,
        uint32 _maxConversionFee,
        IERC20Token _connectorToken,
        uint32 _connectorWeight
    )
        public
        SmartTokenController(_token)
        validAddress(_registry)
        validMaxConversionFee(_maxConversionFee)
    {
        registry = _registry;
        IContractFeatures features = IContractFeatures(registry.addressOf(ContractIds.CONTRACT_FEATURES));

        // initialize supported features（初始化支持特征）
        if (features != address(0))
            features.enableFeatures(FeatureIds.CONVERTER_CONVERSION_WHITELIST, true);

        maxConversionFee = _maxConversionFee;

        if (_connectorToken != address(0))
            addConnector(_connectorToken, _connectorWeight, false);
	}


    // validates a connector token address - verifies that the address belongs to one of the connector tokens（验证连接器令牌地址验证地址属于连接器令牌之一。）
    
    modifier validConnector(IERC20Token _address) {
        require(connectors[_address].isSet);
        _;
	}

	// validates a token address - verifies that the address belongs to one of the convertible tokens（验证令牌地址-验证地址属于可转换令牌之一。）
	
    modifier validToken(IERC20Token _address) {
        require(_address == token || connectors[_address].isSet);
        _;
	}

    // validates maximum conversion fee（验证最大转换费）
    
    modifier validMaxConversionFee(uint32 _conversionFee) {
        require(_conversionFee >= 0 && _conversionFee <= MAX_CONVERSION_FEE);
        _;
	}

  	// validates conversion fee（验证转换费）
  
    modifier validConversionFee(uint32 _conversionFee) {
        require(_conversionFee >= 0 && _conversionFee <= maxConversionFee);
        _;
	}

    // validates connector weight range（验证连接器的宽度范围）
    modifier validConnectorWeight(uint32 _weight) {
        require(_weight > 0 && _weight <= MAX_WEIGHT);
        _;
	}

    // validates a conversion path - verifies that the number of elements is odd and that maximum number of 'hops' is 10（验证转换路径-验证元素的数目是奇数的，并且“跳数”的最大数目是10）
    
    modifier validConversionPath(IERC20Token[] _path) {
        require(_path.length > 2 && _path.length <= (1 + 2 * 10) && _path.length % 2 == 1);
        _;
	}

	// allows execution only when conversions aren't disabled（仅当转换未禁用时才允许执行）

   	 modifier conversionsAllowed {
        	assert(conversionsEnabled);
       		 _;
    	}


    // allows execution by the BancorNetwork contract only（允许仅由BANCORNET契约执行）
    
    modifier bancorNetworkOnly {
        IBancorNetwork bancorNetwork = IBancorNetwork(registry.addressOf(ContractIds.BANCOR_NETWORK));
        require(msg.sender == address(bancorNetwork));
        _;
	}


    /**
        @dev returns the number of connector tokens defined（返回定义的连接器令牌的数量）
        @return number of connector tokens（连接器令牌返回数）
    */
    
    function connectorTokenCount() public view returns (uint16) {
        return uint16(connectorTokens.length);
	}


	/*
        @dev allows the owner to update the contract registry contract address（允许所有者更新合同注册表合同地址）
        @param _registry   address of a contract registry contract（合同登记合同地址）
    	*/
	
    function setRegistry(IContractRegistry _registry)
        public
        ownerOnly
        validAddress(_registry)
        notThis(_registry)
    {
        registry = _registry;
    }


    /*
        @dev allows the owner to update & enable the conversion whitelist contract address
        when set, only addresses that are whitelisted are actually allowed to use the converter
        note that the whitelist check is actually done by the BancorNetwork contract（允许所有者更新和启用转换白名单契约地址当设置时，只允许使用白名单的地址使用转换器。注意，白名单检查实际上是由BANCORNET契约完成的。）
        @param _whitelist    address of a whitelist contract（白名单合同地址）
    */
    function setConversionWhitelist(IWhitelist _whitelist)
        public
        ownerOnly
        notThis(_whitelist)
    {
        conversionWhitelist = _whitelist;
	}


	/*
        @dev allows the manager to update the quick buy path（允许管理员更新快速购买路径）
        @param _path    new quick buy path, see conversion path format in the bancorNetwork contract（新的快速购买路径，参见BANCORNET合同	中的转换路径格式）
    	*/
	
    function setQuickBuyPath(IERC20Token[] _path)
        public
        ownerOnly
        validConversionPath(_path)
    	{
        quickBuyPath = _path;
	}

	/*
        @dev allows the manager to clear the quick buy path（允许管理员清除快速购买路径）
    	*/
    	function clearQuickBuyPath() public ownerOnly {
        	quickBuyPath.length = 0;
   	 }

    /**
        @dev returns the length of the quick buy path array(返回快速购买路径数组的长度)
        @return quick buy path length(返回快速购买路径长度)
    */
    function getQuickBuyPathLength() public view returns (uint256) {
        return quickBuyPath.length;
	}
	
 	/**
        @dev disables the entire conversion functionality
        this is a safety mechanism in case of a emergency
        can only be called by the manager  (禁用整个转换功能。这是一种安全机制，在紧急情况下只能由管理者调用)
        @param _disable true to disable conversions, false to re-enable them (true禁用转换,false重新启用它们)
    */
    function disableConversions(bool _disable) public ownerOrManagerOnly {
        conversionsEnabled = !_disable;
	}

	/**
        @dev updates the current conversion fee
        can only be called by the manager (更新当前转换费只能由经理调用)
        @param _conversionFee new conversion fee, represented in ppm(新的转换费，以PPM表示)
    	*/
    function setConversionFee(uint32 _conversionFee)
        public
        ownerOrManagerOnly
        validConversionFee(_conversionFee)
    {
        emit ConversionFeeUpdate(conversionFee, _conversionFee);
        conversionFee = _conversionFee;
	}

	/*
        @dev given a return amount, returns the amount minus the conversion fee(返回给定的数量，金额减去“归来”的转换 )
        @param _amount      return amount(输出值)
        @param _magnitude   1 for standard conversion, 2 for cross connector conversion（1用于标准转换，2用于交叉连接器转换）
        @return return amount minus conversion fee（退还金额减去换算费）
    	*/
    function getFinalAmount(uint256 _amount, uint8 _magnitude) public view returns (uint256) {
        return safeMul(_amount, (MAX_CONVERSION_FEE - conversionFee) ** _magnitude) / MAX_CONVERSION_FEE ** _magnitude;
	}

 	/**
        @dev defines a new connector for the token can only be called by the owner while the converter is inactive (定义一个新的连接器，令牌只能在转换器不活动时由所有者调用)
        @param _token                  address of the connector token (连接器令牌地址)
        @param _weight                 constant connector weight, represented in ppm, 1-1000000(恒定的连接器费用，以PPM表示，1-100000)
        @param _enableVirtualBalance   true to enable virtual balance for the connector, false to disable it（如果要启用连接器的虚拟余额，请将其禁用）
    	*/
    function addConnector(IERC20Token _token, uint32 _weight, bool _enableVirtualBalance)
        public
        ownerOnly
        inactive
        validAddress(_token)
        notThis(_token)
        validConnectorWeight(_weight)
    {
        require(_token != token && !connectors[_token].isSet && totalConnectorWeight + _weight <= MAX_WEIGHT); // validate input

        connectors[_token].virtualBalance = 0;
        connectors[_token].weight = _weight;
        connectors[_token].isVirtualBalanceEnabled = _enableVirtualBalance;
        connectors[_token].isPurchaseEnabled = true;
        connectors[_token].isSet = true;
        connectorTokens.push(_token);
        totalConnectorWeight += _weight;
    }

    /**
        @dev updates one of the token connectors can only be called by the owner（更新其中一个令牌连接器 只能由所有者调用）
        @param _connectorToken         address of the connector token（连接器令牌地址）
        @param _weight                 constant connector weight, represented in ppm, 1-1000000（恒定的连接器费用，以PPM表示，1-100000）
        @param _enableVirtualBalance   true to enable virtual balance for the connector, false to disable it（如果要启用连接器的虚拟余额，请将其禁用）
        @param _virtualBalance         new connector's virtual balance（新连接器的虚拟平衡）
    */
    function updateConnector(IERC20Token _connectorToken, uint32 _weight, bool _enableVirtualBalance, uint256 _virtualBalance)
        public
        ownerOnly
        validConnector(_connectorToken)
        validConnectorWeight(_weight)
    {
        Connector storage connector = connectors[_connectorToken];
        require(totalConnectorWeight - connector.weight + _weight <= MAX_WEIGHT); // validate input（验证输入）

        totalConnectorWeight = totalConnectorWeight - connector.weight + _weight;
        connector.weight = _weight;
        connector.isVirtualBalanceEnabled = _enableVirtualBalance;
        connector.virtualBalance = _virtualBalance;
	}

	/**
        @dev disables purchasing with the given connector token in case the connector token got compromised
        can only be called by the owner
        note that selling is still enabled regardless of this flag and it cannot be disabled by the owner
 	（如果连接器令牌受损，则使用给定的连接器令牌进行购买的isables只能由所有者调用，因为销售仍然启用，而不管这个标志是什么，并且所有者不能禁用它）
        @param _connectorToken  connector token contract address（连接器令牌合同地址）
        @param _disable         true to disable the token, false to re-enable it（true 禁用令牌，false 使其重新启用）
   	 */
    function disableConnectorPurchases(IERC20Token _connectorToken, bool _disable)
        public
        ownerOnly
        validConnector(_connectorToken)
    {
        connectors[_connectorToken].isPurchaseEnabled = !_disable;
	}

	/**
        @dev returns the connector's virtual balance if one is defined, otherwise returns the actual balance（如果定义了一个连接器，则返回连接器的虚拟余额，否则返回实际余额）
        @param _connectorToken  connector token contract address（连接器令牌合同地址）
        @return connector balance（连接器平衡）
    	*/
    function getConnectorBalance(IERC20Token _connectorToken)
        public
        view
        validConnector(_connectorToken)
        returns (uint256)
    {
        Connector storage connector = connectors[_connectorToken];
        return connector.isVirtualBalanceEnabled ? connector.virtualBalance : _connectorToken.balanceOf(this);
	}

	/**
        @dev returns the expected return for converting a specific amount of _fromToken to _toToken（返回将特定金额从_fromToken令牌转换到 _toToken令牌的预期回报）
        @param _fromToken  ERC20 token to convert from （Erc20令牌转换）
        @param _toToken    ERC20 token to convert to（Erc20令牌获得）
        @param _amount     amount to convert, in fromToken（从FoT中转换的金额）
        @return expected conversion return amount（返回预期转换收益金额）
    	*/
    function getReturn(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount) public view returns (uint256) {
        require(_fromToken != _toToken); // validate input（验证输入）

        // conversion between the token and one of its connectors（令牌与其一个连接器之间的转换）
        if (_toToken == token)
            return getPurchaseReturn(_fromToken, _amount);
        else if (_fromToken == token)
            return getSaleReturn(_toToken, _amount);

        // conversion between 2 connectors（2个连接器之间的转换）
        return getCrossConnectorReturn(_fromToken, _toToken, _amount);
	}

	/**
        @dev returns the expected return for buying the token for a connector token（返回购买连接器令牌令牌的预期收益）
        @param _connectorToken  connector token contract address（连接器令牌合同地址）
        @param _depositAmount   amount to deposit (in the connector token)（存款金额（在连接器令牌中））
        @return expected purchase return amount（返回预期购买回报金额）
    	*/
    function getPurchaseReturn(IERC20Token _connectorToken, uint256 _depositAmount)
        public
        view
        active
        validConnector(_connectorToken)
        returns (uint256)
    {
        Connector storage connector = connectors[_connectorToken];
        require(connector.isPurchaseEnabled); // validate input

        uint256 tokenSupply = token.totalSupply();
        uint256 connectorBalance = getConnectorBalance(_connectorToken);
        IBancorFormula formula = IBancorFormula(registry.addressOf(ContractIds.BANCOR_FORMULA));
        uint256 amount = formula.calculatePurchaseReturn(tokenSupply, connectorBalance, connector.weight, _depositAmount);

        // return the amount minus the conversion fee（退还金额减去转换费）
        return getFinalAmount(amount, 1);
	}

    /**
        @dev returns the expected return for selling the token for one of its connector tokens（返回为其连接器令牌之一出售令牌的预期收益）
        @param _connectorToken  connector token contract address（连接器令牌合同地址）
        @param _sellAmount      amount to sell (in the smart token)（出售金额（在智能令牌中））
        @return expected sale return amount（预期销售返还金额）
    */
    function getSaleReturn(IERC20Token _connectorToken, uint256 _sellAmount)
        public
        view
        active
        validConnector(_connectorToken)
        returns (uint256)
    {
        Connector storage connector = connectors[_connectorToken];
        uint256 tokenSupply = token.totalSupply();
        uint256 connectorBalance = getConnectorBalance(_connectorToken);
        IBancorFormula formula = IBancorFormula(registry.addressOf(ContractIds.BANCOR_FORMULA));
        uint256 amount = formula.calculateSaleReturn(tokenSupply, connectorBalance, connector.weight, _sellAmount);

        // return the amount minus the conversion fee（退还金额减去转换费）
        return getFinalAmount(amount, 1);
	}

 	/**
        @dev returns the expected return for selling one of the connector tokens for another connector token（返回购买另一个tokens的预期收益）
        @param _fromConnectorToken  contract address of the connector token to convert from（要转换的连接器令牌的合同地址）
        @param _toConnectorToken    contract address of the connector token to convert to（转换为的连接器令牌的转换地址）
        @param _sellAmount          amount to sell (in the from connector token)（销售金额（在连接器令牌中））
        @return expected sale return amount (in the to connector token)销售金额（在连接器令牌中）
   	 */
    function getCrossConnectorReturn(IERC20Token _fromConnectorToken, IERC20Token _toConnectorToken, uint256 _sellAmount)
        public
        view
        active
        validConnector(_fromConnectorToken)
        validConnector(_toConnectorToken)
        returns (uint256)
    {
        Connector storage fromConnector = connectors[_fromConnectorToken];
        Connector storage toConnector = connectors[_toConnectorToken];
        require(toConnector.isPurchaseEnabled); // validate input

        uint256 fromConnectorBalance = getConnectorBalance(_fromConnectorToken);
        uint256 toConnectorBalance = getConnectorBalance(_toConnectorToken);

        IBancorFormula formula = IBancorFormula(registry.addressOf(ContractIds.BANCOR_FORMULA));
        uint256 amount = formula.calculateCrossConnectorReturn(fromConnectorBalance, fromConnector.weight, toConnectorBalance, toConnector.weight, _sellAmount);

        // return the amount minus the conversion fee
        // the fee is higher (magnitude = 2) since cross connector conversion equals 2 conversions (from / to the smart token)
        return getFinalAmount(amount, 2);
	}

 	/**
        @dev converts a specific amount of _fromToken to _toToken （转换一个特定的量_fromToken 到 _toToken）
        @param _fromToken  ERC20 token to convert from（要转换的连接器令牌）
        @param _toToken    ERC20 token to convert to（转换为的令牌）
        @param _amount     amount to convert, in fromToken（从FoT中转换的金额）
        @param _minReturn  if the conversion results in an amount smaller than the minimum return - it is cancelled, must be nonzero（如果转换结果小于最小返回量，则被取消，必须为非零）
        @return conversion return amount（转换收益金额）
    	*/
    function convertInternal(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn)
        public
        bancorNetworkOnly
        conversionsAllowed
        greaterThanZero(_minReturn)
        returns (uint256)
    {
        require(_fromToken != _toToken); // validate input

        // conversion between the token and one of its connectors（令牌与其一个连接器之间的转换）
        if (_toToken == token)
            return buy(_fromToken, _amount, _minReturn);
        else if (_fromToken == token)
            return sell(_toToken, _amount, _minReturn);

        // conversion between 2 connectors（2个连接器之间的转换）
        uint256 amount = getCrossConnectorReturn(_fromToken, _toToken, _amount);
        // ensure the trade gives something in return and meets the minimum requested amount（确保贸易给予回报并满足最低要求的数量）
        require(amount != 0 && amount >= _minReturn);

        // update the source token virtual balance if relevant（如果相关的话更新源令牌虚拟余额）
        Connector storage fromConnector = connectors[_fromToken];
        if (fromConnector.isVirtualBalanceEnabled)
            fromConnector.virtualBalance = safeAdd(fromConnector.virtualBalance, _amount);

        // update the target token virtual balance if relevant（如果相关，更新目标令牌虚拟余额）
        Connector storage toConnector = connectors[_toToken];
        if (toConnector.isVirtualBalanceEnabled)
            toConnector.virtualBalance = safeSub(toConnector.virtualBalance, amount);

        // ensure that the trade won't deplete the connector balance（确保交易不会耗尽连接器的平衡）
        uint256 toConnectorBalance = getConnectorBalance(_toToken);
        assert(amount < toConnectorBalance);

        // transfer funds from the caller in the from connector token（从连接器令牌中向呼叫者转移资金）
        assert(_fromToken.transferFrom(msg.sender, this, _amount));
        // transfer funds to the caller in the to connector token（将资金转移到接线员令牌中的呼叫者）
        // the transfer might fail if the actual connector balance is smaller than the virtual balance（如果实际连接器平衡小于虚拟平衡，传输可能会失败。）
        assert(_toToken.transfer(msg.sender, amount));

        // calculate conversion fee and dispatch the conversion event（计算转换费用并发送转换事件）
        // the fee is higher (magnitude = 2) since cross connector conversion equals 2 conversions (from / to the smart token)（由于交叉连接器转换等于2（从/到智能令牌），费用更高（幅度＝2））
        uint256 feeAmount = safeSub(amount, getFinalAmount(amount, 2));
        dispatchConversionEvent(_fromToken, _toToken, _amount, amount, feeAmount);

        // dispatch price data updates for the smart token / both connectors（智能令牌/两个连接器的调度价格数据更新）
        emit PriceDataUpdate(_fromToken, token.totalSupply(), getConnectorBalance(_fromToken), fromConnector.weight);
        emit PriceDataUpdate(_toToken, token.totalSupply(), getConnectorBalance(_toToken), toConnector.weight);
        return amount;
    	}


