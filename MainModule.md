## Main Code

---

## IF YOU COPY AND PASTE, BE SURE TO INCLUDE THE DEPENDENCIES

---

```lua
--[[ CommerceHub.lua

	A wrapper around Roblox's Default `MarketplaceService`, designed to simplify
	gameplay purchase systems while providing a clean, extensible API.

	CommerceHub goes beyond simple purchase prompts by including built-in
	validation, caching, asynchronous workflows, signals, rate limiting,
	and lifecycle management, making it suitable for both small projects
	and larger experiences.

	--// Features

	*  Gamepass & Developer Product purchase wrappers
	*  Built-in gamepass gifting system
	*  Promise-based asynchronous API
	*  Signal & callback support
	*  Product information and price caching
	*  User, asset, and receipt validation
	*  Configurable rate limiting
	*  Purchase ticket tracking
	*  Automatic cleanup and lifecycle management
	*  Regional price support
	*  Clean, developer-friendly API
	
	---

	--// Example

	```lua
	local CommerceHub = require(path.To.CommerceHub)

	local Commerce = CommerceHub.new()
	Commerce:Init()

	Commerce:PromptGamepassPurchaseAsync(player.UserId, GAMEPASS_ID)
  	  :andThen(function()
  	      print("Purchase prompt opened!")
  	  end)
    	:catch(function(err)
    	    warn(err)
   	 end)
	```
	
	---
	
	--// Q & A

	Q. Why CommerceHub?
	A. Instead of interacting directly with `MarketplaceService` every time, CommerceHub provides a centralized API that handles common concerns automatically, allowing you to focus on building your game rather than rewriting commerce logic.

	---

	--// Goals

	* Reduce repetitive MarketplaceService code.
	* Provide a safer and more consistent purchasing API.
	* Encourage scalable, maintainable commerce systems.
	* Make advanced commerce features accessible through a simple interface.

	---

	Built with love an passion for other Roblox developers. <3

	---
	
	--// Changlogs
	
	---

	GitHub (repository):
	https://github.com/draxxordev/CommerceHub

	By @Draxxor
]]--

--// Services
local MarketplaceService = game:GetService("MarketplaceService")
local Players = game:GetService("Players")

--// Dependencies
--//  !! MAKE SURE YOU HAVE THESE !! \\ --
local ProfileStore = require("@self/ProfileStore")
local Promise = require("@self/Promise")
local Signal = require("@self/Signal")
local Maid = require("@self/Maid")

--// Types
export type Callback = (...any) -> ...any

export type SignalObject = Signal.Signal<Callback>

export type Ticket = {
	Info: any,
	Owners: { number },
	Job: string,
}

export type PurchaseData = {
	UserId: number,
	AssetId: number,
	AssetType: string,
	Timestamp: number,
}

export type Receipt = {
	UserId: number,
	ProductId: number,
	Timestamp: number,
}

export type CachedPriceInfo = {
	UserId: number,
	ProductId: number,
	Success: boolean,
	Price: number,
}

export type RateData = {
	LastCall: number,
	CallCount: number
}

export type Gift = {
	FromUserId: number,
	GamepassId: number,
	Timestamp: number,
}

--// Classes
local MainStore: ProfileStore.ProfileStore<any> = ProfileStore.New("CommerceHubStore")

--// Variables
local PlayerTickets: { [number]: Ticket } = {}
local ProductInfoCache: { [number]: any } = {}
local PriceCache: { [string]: CachedPriceInfo } = {}
local PurchaseListeners: { Callback } = {}
local GamepassListeners: { Callback } = {}
local ProductListeners: { Callback } = {}
local TicketCounter: number  = 0

--// Rate limiting
local UserRateLimits: { [number]: RateData } = {}

--// Validation
local ReceiptHandlers: { [number]: Callback } = {}
local ProcessedReceipts: { [string]: boolean } = {}
local PendingReceipts: { [number]: Receipt } = {}
local ValidatedPurchases: { [string]: boolean } = {}

--// Extras
local PendingGifts: { [number]: Gift } = {}
local ProcessedGifts: { [number]: { [number]: boolean } } = {}

--// Constants
local cacheExpiry = 300 -- 5 minutes
local rateLimitCalls = 10 -- max calls per window
local rateLimitWindow = 5 -- seconds
local minAssetId = 1
local receiptExpiry = 600 -- 10 minutes

--// Signals
local PurchaseSignal: SignalObject = Signal.new()
local GamepassSignal: SignalObject = Signal.new()
local ProductSignal: SignalObject = Signal.new()
local GiftReceivedSignal: SignalObject = Signal.new()

--// Private Functions

local function getCacheKey(userId, productId)
	return userId .. ":" .. productId
end

local function getProductInfoAsync(assetId)
	return Promise.new(function(resolve, reject)
		local success, info = pcall(function()
			return MarketplaceService:GetProductInfoAsync(assetId, Enum.InfoType.Product)
		end)

		if success then
			resolve(info)
		else
			reject("Failed to fetch product info: " .. tostring(info))
		end
	end)
end

local function getUserOwnedAssets(userId, assetTypeId)
	return Promise.new(function(resolve, reject)
		local assets = {}
		local cursor = ""
		local success = true

		while success do
			local ok, result = pcall(function()
				return MarketplaceService:GetUserAssetsAsync(userId, assetTypeId, Enum.SortOrder.Asc, 100, cursor)
			end)

			if not ok then
				reject("Failed to fetch user assets: " .. tostring(result))
				return
			end

			for _, asset in ipairs(result:GetAssets()) do
				table.insert(assets, asset.Id)
			end

			cursor = result:GetNextPageCursor()
			if cursor == "" then
				success = false
			end
		end

		resolve(assets)
	end)
end

--[[ Validate User ID ]]--
local function validateUserId(userId)
	return typeof(userId) == "number"
		and userId > 0
		and userId % 1 == 0
end

--[[ Validate Asset ID ]]--
local function validateAssetId(assetId)
	return type(assetId) == "number" and assetId >= minAssetId
end

--[[ Check Rate Limit ]]--
local function checkRateLimit(userId)
	if not UserRateLimits[userId] then
		UserRateLimits[userId] = {
			LastCall = 0,
			CallCount = 0,
		}
	end

	local limit = UserRateLimits[userId]
	local now = tick()

	-- Reset the counter if we are outside the window
	if now - limit.LastCall > rateLimitWindow then
		limit.CallCount = 0
		limit.LastCall = now
	end

	-- Check if the user has exceeded the limit
	if limit.CallCount >= rateLimitCalls then
		return false, "Rate limit exceeded"
	end

	limit.CallCount = limit.CallCount + 1
	
	return true, nil
end

local function validateRate(userId, reject)
	-- Check the rate limit for this user to prevent repeated calls
	local ok, err = checkRateLimit(userId)

	if not ok then
		return Promise.reject(err)
	end
end

--[[ Track Purchase Receipt ]]--
local function trackReceipt(receiptId, userId, productId)
	PendingReceipts[receiptId] = {
		UserId = userId,
		ProductId = productId,
		Timestamp = os.time(),
	}

	-- Auto cleanup the receipt after timeout
	task.delay(receiptExpiry, function()
		PendingReceipts[receiptId] = nil
	end)
end

--[[ Validate Receipt ]]--
local function validateReceipt(receiptId, userId, productId)
	local receipt = PendingReceipts[receiptId]
	if not receipt then
		return false, "Receipt not found"
	end

	if receipt.UserId ~= userId then
		return false, "User ID mismatch"
	end

	if receipt.ProductId ~= productId then
		return false, "Product ID mismatch"
	end

	-- Mark the purchase as validated
	local purchaseKey = userId .. "-" .. productId
	
	ValidatedPurchases[purchaseKey] = true

	-- Then we cleanup the reciept after timeout
	PendingReceipts[receiptId] = nil

	return true, nil
end

local function attemptPurchaseAsync(userId, productId)
	return Promise.new(function(resolve, reject)
		-- Validate arguments
		if not validateUserId(userId) then
			reject("Invalid user ID")
			return
		end

		if not validateAssetId(productId) then
			reject("Invalid product ID")
			return
		end

		validateRate(userId, reject)

		local player = Players:GetPlayerByUserId(userId)
		if not player then
			reject("Player not found")
			return
		end

		local success, result = pcall(function()
			MarketplaceService:PromptProductPurchase(player, productId)
		end)

		if success then
			resolve("Purchase prompt shown")
		else
			reject("Failed to prompt purchase: " .. tostring(result))
		end
	end)
end

local function fireProductPurchased(userId, productId, receiptId)
	local data = {
		UserId = userId,
		ProductId = productId,
		ReceiptId = receiptId,
		Timestamp = os.time(),
	}

	ProductSignal:Fire(data)

	-- Fire all the callbacks
	for _, callback in ipairs(ProductListeners) do
		task.spawn(callback, data)
	end
end

local function fireGamepassAcquired(userId, gamepassId)
	local data = {
		UserId = userId,
		GamepassId = gamepassId,
		Timestamp = os.time(),
	}

	GamepassSignal:Fire(data)

	-- Fire all the callbacks
	for _, callback in ipairs(GamepassListeners) do
		task.spawn(callback, data)
	end
end

local function firePurchaseComplete(userId, assetId, assetType)
	local data = {
		UserId = userId,
		AssetId = assetId,
		AssetType = assetType,
		Timestamp = os.time(),
	}

	PurchaseSignal:Fire(data)

	-- Fire all the callbacks
	for _, callback in ipairs(PurchaseListeners) do
		task.spawn(callback, data)
	end
end

local function processReceipt(receiptInfo)
	local purchaseId = receiptInfo.PurchaseId
	local userId = receiptInfo.PlayerId
	local productId = receiptInfo.ProductId

	-- Already granted
	if ProcessedReceipts[purchaseId] then
		return Enum.ProductPurchaseDecision.PurchaseGranted
	end

	local player = Players:GetPlayerByUserId(userId)

	-- The player left, retry later
	if not player then
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end

	local handler = ReceiptHandlers[productId]

	if not handler then
		warn("No receipt handler for product:", productId)
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end


	local success, err = pcall(function()
		handler(player, receiptInfo)
	end)

	if not success then
		warn("Failed granting product:", productId, err)
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end

	ProcessedReceipts[purchaseId] = true

	fireProductPurchased(
		userId,
		productId,
		purchaseId
	)

	return Enum.ProductPurchaseDecision.PurchaseGranted
end

local function setupGamepassListener()
	MarketplaceService.PromptGamePassPurchaseFinished:Connect(
		function(player, gamepassId, purchased)

			if purchased then

				fireGamepassAcquired(
					player.UserId,
					gamepassId
				)

			end
		end
	)
end

--// Public Functions

local CommerceHub = {}
CommerceHub.__index = CommerceHub

--[[ new
	Main constructor for the module
]]
function CommerceHub.new()
	return setmetatable({}, CommerceHub)
end

--[[ Destroy
	Cleans up the module
	Cleans up all resources and connections
]]
function CommerceHub:Destroy()
	table.clear(PlayerTickets)
	table.clear(ProductInfoCache)
	table.clear(PriceCache)

	table.clear(UserRateLimits)

	table.clear(PendingReceipts)
	table.clear(ValidatedPurchases)

	table.clear(PendingGifts)
	table.clear(ProcessedGifts)

	table.clear(PurchaseListeners)
	table.clear(GamepassListeners)
	table.clear(ProductListeners)

	PurchaseSignal:Destroy()
	GamepassSignal:Destroy()
	ProductSignal:Destroy()
	GiftReceivedSignal:Destroy()
end

--[[ Init
	Sets up the modules cleanup automatically
]]
function CommerceHub:Init()
	--[[ Setup the main gamepass listener
]]--
	setupGamepassListener()
	
	--[[ Connection to ProcessReceipt
]]--
	MarketplaceService.ProcessReceipt = processReceipt
	
	--[[ Connection to player join - Load gifts
	Auto-loads pending gifts when a player joins
]]--
	Players.PlayerAdded:Connect(function(player)
		self:LoadPlayerGifts(player.UserId):catch(function(err)
			warn("Failed to load gifts for player " .. player.UserId .. ": " .. tostring(err))
		end)
	end)

--[[ Connection to player removal
	Auto-cleanup when players leave
]]--
	Players.PlayerRemoving:Connect(function(player)
		self:Cleanup(player.UserId)

		-- We also need to cleanup any potentially pending gifts
		PendingGifts[player.UserId] = nil
		ProcessedGifts[player.UserId] = nil
	end)
	
--[[ Connections to game close
	Auto-cleanup when the game is closing
]]--
	game:BindToClose(function()
		self:Destroy()
	end)
end

--[[ GetProductInfoAsync
	Fetches product information with caching
	@param assetId: The asset ID to fetch info for
	@return Promise that resolves with ProductInfo
]]--
function CommerceHub:GetProductInfoAsync(assetId)
	if not validateAssetId(assetId) then
		return Promise.reject("Invalid asset ID")
	end

	if ProductInfoCache[assetId] then
		return Promise.resolve(ProductInfoCache[assetId])
	end

	return getProductInfoAsync(assetId):andThen(function(info)
		ProductInfoCache[assetId] = info
		-- Cache for 5 minutes
		task.delay(cacheExpiry, function()
			ProductInfoCache = nil
		end)
		
		return info
	end)
end

--[[ IsGamepassOwnedAsync
	Checks if a user owns a specific gamepass
	@param userId: The user ID to check
	@param gamepassId: The gamepass asset ID
	@return Promise that resolves with boolean
]]--
function CommerceHub:IsGamepassOwnedAsync(userId, gamepassId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	if not validateAssetId(gamepassId) then
		return Promise.reject("Invalid gamepass ID")
	end

	validateRate(userId, Promise.reject)

	return Promise.new(function(resolve, reject)
		local success, owned = pcall(function()
			return MarketplaceService:UserOwnsGamePassAsync(userId, gamepassId)
		end)

		if success then
			resolve(owned)
		else
			reject("Failed to check gamepass ownership: " .. tostring(owned))
		end
	end)
end

--[[ IsProductOwnedAsync
	Checks if a user owns a specific developer product
	@param userId: The user ID to check
	@param productId: The product asset ID
	@return Promise that resolves with boolean
]]--
function CommerceHub:IsProductOwnedAsync(userId, productId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	if not validateAssetId(productId) then
		return Promise.reject("Invalid product ID")
	end

	validateRate(userId, Promise.reject)

	return getUserOwnedAssets(userId, Enum.AssetType.DevProduct):andThen(function(assets)
		return table.find(assets, productId) ~= nil
	end)
end

--[[ GiftGamepassAsync
	Gifts a gamepass to another player (stored in ProfileStore)
	@param fromUserId: The user ID gifting the gamepass
	@param toUserId: The user ID receiving the gamepass
	@param gamepassId: The gamepass asset ID
	@return Promise that resolves when gift is saved
]]--
function CommerceHub:GiftGamepassAsync(fromUserId, toUserId, gamepassId)
	-- We need to validate all arguments
	if not validateUserId(fromUserId) then
		return Promise.reject("Invalid from user ID")
	end

	if not validateUserId(toUserId) then
		return Promise.reject("Invalid to user ID")
	end

	if not validateAssetId(gamepassId) then
		return Promise.reject("Invalid gamepass ID")
	end

	if fromUserId == toUserId then
		return Promise.reject("Cannot gift to self")
	end

	validateRate(fromUserId, Promise.reject)

	return Promise.new(function(resolve, reject)
		-- Then we initialize pending gifts table if needed
		if not PendingGifts[toUserId] then
			PendingGifts[toUserId] = {}
		end

		local gift = {
			FromUserId = fromUserId,
			GamepassId = gamepassId,
			Timestamp = os.time(),
		}

		-- Add to pending gifts
		table.insert(PendingGifts[toUserId], gift)

		-- Save to ProfileStore asynchronously
		task.spawn(function()
			local success, result = pcall(function()
				local store = MainStore:StartSessionAsync("gifts-" .. toUserId)
				if not store then
					reject("Failed to load store")
					return
				end

				if not store.Data.Gifts then
					store.Data.Gifts = {}
				end

				table.insert(store.Data.Gifts, gift)
				store:EndSession()
			end)

			if not success then
				reject("Failed to save gift: " .. tostring(result))
			else
				-- Mark the gift as processed
				if not ProcessedGifts[toUserId] then
					ProcessedGifts[toUserId] = {}
				end
				
				ProcessedGifts[toUserId][gamepassId] = true

				resolve(gift)
			end
		end)
	end)
end

--[[ PromptProductPurchaseAsync
	Prompts a player to purchase a developer product
	@param userId: The user ID to prompt
	@param productId: The product asset ID
	@param expectedPrice: Optional expected price in Robux
	@return Promise that resolves when purchase is prompted
]]--
function CommerceHub:PromptProductPurchaseAsync(userId, productId)
	return attemptPurchaseAsync(userId, productId)
end

--[[ PromptGamepassPurchaseAsync
	Prompts a player to purchase a gamepass
	@param userId: The user ID to prompt
	@param gamepassId: The gamepass asset ID
	@return Promise that resolves when purchase is prompted
]]--
function CommerceHub:PromptGamepassPurchaseAsync(userId, gamepassId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	if not validateAssetId(gamepassId) then
		return Promise.reject("Invalid gamepass ID")
	end

	validateRate(userId, Promise.reject)

	return Promise.new(function(resolve, reject)
		local player = Players:GetPlayerByUserId(userId)
		if not player then
			reject("Player not found")
			return
		end

		local success, result = pcall(function()
			MarketplaceService:PromptGamePassPurchase(player, gamepassId)
		end)

		if success then
			resolve("Gamepass purchase prompt shown")
		else
			reject("Failed to prompt gamepass purchase: " .. tostring(result))
		end
	end)
end

--[[  RegisterProductHandler
	Registers a handler for a product purchase
	@param productId: The product asset ID
	@param callback: The handler function to handle the purchase
]]--
function CommerceHub:RegisterProductHandler(productId, callback)
	-- Validate the arguments
	if not validateAssetId(productId) then
		error("Invalid product ID")
	end

	if type(callback) ~= "function" then
		error("Callback must be a function")
	end

	ReceiptHandlers[productId] = callback
end

--[[ GetUserGamepassesAsync
	Gets all gamepasses owned by a user
	@param userId: The user ID to check
	@return Promise that resolves with table of gamepass IDs
]]--
function CommerceHub:GetUserGamepassesAsync(userId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	-- Check the rate limit for this user
	local allowed, err = checkRateLimit(userId)
	if not allowed then
		return Promise.reject(err)
	end

	return getUserOwnedAssets(userId, Enum.AssetType.GamePass)
end

--[[ GetProductPriceAsync
	Gets the price of a developer product with caching
	@param userId: The user ID (for regional pricing)
	@param productId: The product asset ID
	@return Promise that resolves with price in Robux
]]--
function CommerceHub:GetProductPriceAsync(userId, productId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	if not validateAssetId(productId) then
		return Promise.reject("Invalid product ID")
	end

	validateRate(userId, Promise.reject)

	local cacheKey = getCacheKey(userId, productId)

	if PriceCache[cacheKey] then
		return Promise.resolve(PriceCache[cacheKey].Price or 0)
	end

	return Promise.new(function(resolve, reject)
		local player = Players:GetPlayerByUserId(userId)
		if not player then
			reject("Player not found")
			return
		end

		local success, price = pcall(function()
			return MarketplaceService:GetProductPrice(productId)
		end)

		if success then
			PriceCache[cacheKey] = {
				UserId = userId,
				ProductId = productId,
				Success = true,
				Price = price,
			}
			
			-- We then setup a clean expiry cache for this entry with a duration of 5 minutes
			task.delay(cacheExpiry, function()
				PriceCache[cacheKey] = nil
			end)
			resolve(price)
		else
			reject("Failed to get product price: " .. tostring(price))
		end
	end)
end

--[[ PurchaseWithSignal
	Prompts a purchase and returns a signal that fires when complete
	@param userId: The user ID to prompt
	@param productId: The product asset ID
	@return Signal that fires with purchase result
]]--
function CommerceHub:PurchaseWithSignal(userId, productId)
	local signal = Signal.new()

	attemptPurchaseAsync(userId, productId):andThen(function()
		signal:Fire(true)
	end):catch(function()
		signal:Fire(false)
	end)

	return signal
end

--[[ CreatePurchaseTicket
	Creates a ticket for tracking purchases with auto-cleanup
	@param userId: The user ID making the purchase
	@param info: Custom info to store
	@return Ticket that auto-cleans up after timeout
]]--
function CommerceHub:CreatePurchaseTicket(userId, info)
	if not validateUserId(userId) then
		error("Invalid user ID")
	end

	TicketCounter = TicketCounter + 1
	local jobId = game.JobId .. "-" .. TicketCounter -- Unique ticket ID for this purchase

	local ticket = {
		Info = info,
		Owners = { userId },
		Job = jobId,
	}

	PlayerTickets[ticket.Job] = ticket

	-- Auto cleanup after 1 hour
	task.delay(3600, function()
		PlayerTickets[ticket.Job] = nil
	end)

	return ticket
end

--[[ GetTicket
	Retrieves a ticket by its job ID
	@param jobId: The ticket job ID
	@return Ticket or nil
]]--
function CommerceHub:GetTicket(jobId)
	return PlayerTickets[jobId]
end

--[[ ValidatePurchaseReceipt
	Validates a purchase receipt
	@param receiptId: The receipt ID from MarketplaceService
	@param userId: The user ID who made the purchase
	@param productId: The product ID that was purchased
	@return Promise that resolves with validation result
]]--
function CommerceHub:ValidatePurchaseReceipt(receiptId, userId, productId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	if not validateAssetId(productId) then
		return Promise.reject("Invalid product ID")
	end

	return Promise.new(function(resolve, reject)
		if not receiptId or type(receiptId) ~= "string" then
			reject("Invalid receipt ID")
			return
		end

		local success, result = validateReceipt(receiptId, userId, productId)
		if success then
			resolve(result)
		else
			reject(result)
		end
	end)
end

--[[ ClearCache
	Clears all cached data
]]--
function CommerceHub:ClearCache()
	table.clear(ProductInfoCache)
	table.clear(PriceCache)
end

--[[ Cleanup
	Removes all player data and connections
	@param userId: The user ID to cleanup
]]--
function CommerceHub:Cleanup(userId)
	if not validateUserId(userId) then
		error("Invalid user ID")
	end

	for jobId, ticket in pairs(PlayerTickets) do
		if table.find(ticket.Owners, userId) then
			PlayerTickets[jobId] = nil
		end
	end

	-- Clear rate limit data
	UserRateLimits[userId] = nil
end

--[[ ListenToPurchases
	Connects a callback to all purchases (gamepasses and products)
	@param callback: Function(data) to call on purchase
	@return Connection object with Disconnect method
]]--
function CommerceHub:ListenToPurchases(callback)
	if type(callback) ~= "function" then
		error("Callback must be a function")
	end

	table.insert(PurchaseListeners, callback)

	local connection = {
		Connected = true,
		Disconnect = function(self)
			if self.Connected then
				local index = table.find(PurchaseListeners, callback)
				if index then
					table.remove(PurchaseListeners, index)
				end
				
				self.Connected = false
			end
		end,
	}

	return connection
end

--[[ ListenToGamepasses
	Connects a callback to gamepass acquisitions
	@param callback: Function(data) to call when gamepass is acquired
	@return Connection object with Disconnect method
]]--
function CommerceHub:ListenToGamepasses(callback)
	if type(callback) ~= "function" then
		error("Callback must be a function")
	end

	table.insert(GamepassListeners, callback)

	local connection = {
		Connected = true,
		Disconnect = function(self)
			if self.Connected then
				local index = table.find(GamepassListeners, callback)
				if index then
					table.remove(GamepassListeners, index)
				end
				self.Connected = false
			end
		end,
	}

	return connection
end

--[[ ListenToProductPurchases
	Connects a callback to product purchases
	@param callback: Function(data) to call when product is purchased
	@return Connection object with Disconnect method
]]--
function CommerceHub:ListenToProductPurchases(callback)
	if type(callback) ~= "function" then
		error("Callback must be a function")
	end

	table.insert(ProductListeners, callback)

	local connection = {
		Connected = true,
		Disconnect = function(self)
			if self.Connected then
				local index = table.find(ProductListeners, callback)
				if index then
					table.remove(ProductListeners, index)
				end
				self.Connected = false
			end
		end,
	}

	return connection
end

--[[ OnPurchaseComplete
	Returns a signal that fires when any purchase completes
	@return Signal that fires with purchase data
]]--
function CommerceHub:OnPurchaseComplete()
	return PurchaseSignal
end

--[[ OnGamepassAcquired
	Returns a signal that fires when a gamepass is acquired
	@return Signal that fires with gamepass data
]]--
function CommerceHub:OnGamepassAcquired()
	return GamepassSignal
end

--[[ OnProductPurchased
	Returns a signal that fires when a product is purchased
	@return Signal that fires with product data
]]--
function CommerceHub:OnProductPurchased()
	return ProductSignal
end

--[[ PromptGamepassPurchaseWithCallback
	Prompts a gamepass purchase and fires callback on completion
	@param userId: The user ID to prompt
	@param gamepassId: The gamepass asset ID
	@param onComplete: Callback function(success, userId, gamepassId, error)
	@return Maid for cleanup
]]--
function CommerceHub:PromptGamepassPurchaseWithCallback(userId, gamepassId, onComplete)
	if type(onComplete) ~= "function" then
		error("onComplete must be a function")
	end

	local maid = Maid.new()

	local promise = self:PromptGamepassPurchaseAsync(userId, gamepassId)

	promise:andThen(function()
		fireGamepassAcquired(userId, gamepassId)
		onComplete(true, userId, gamepassId)
	end):catch(function(err)
		onComplete(false, userId, gamepassId, err)
	end)

	return maid
end

--[[ PromptProductPurchaseWithCallback
	Prompts a product purchase and fires callback on completion
	@param userId: The user ID to prompt
	@param productId: The product asset ID
	@param onComplete: Callback function(success, userId, productId, error)
	@return Promise that resolves on completion
]]--
function CommerceHub:PromptProductPurchaseWithCallback(userId, productId, onComplete)
	-- Validate parameters
	if type(onComplete) ~= "function" then
		error("onComplete must be a function")
	end


	return self:PromptProductPurchaseAsync(
		userId,
		productId
	):andThen(function()
		onComplete(
			true,
			userId,
			productId
		)
	end):catch(function(err)
		onComplete(
			false,
			userId,
			productId,
			err
		)
	end)
end

--[[ UnlistenToPurchases
	Removes a specific callback from purchase listeners
	@param callback: The callback function to remove
]]--
function CommerceHub:UnlistenToPurchases(callback)
	local index = table.find(PurchaseListeners, callback)
	if index then
		table.remove(PurchaseListeners, index)
	end
end

--[[ UnlistenToGamepasses
	Removes a specific callback from gamepass listeners
	@param callback: The callback function to remove
]]--
function CommerceHub:UnlistenToGamepasses(callback)
	local index = table.find(GamepassListeners, callback)
	if index then
		table.remove(GamepassListeners, index)
	end
end

--[[ UnlistenToProductPurchases
	Removes a specific callback from product listeners
	@param callback: The callback function to remove
]]--
function CommerceHub:UnlistenToProductPurchases(callback)
	local index = table.find(ProductListeners, callback)
	if index then
		table.remove(ProductListeners, index)
	end
end

--[[ ClearAllListeners
	Clears all purchase listeners (use with caution)
]]--
function CommerceHub:ClearAllListeners()
	table.clear(PurchaseListeners)
	table.clear(GamepassListeners)
	table.clear(ProductListeners)
end

--[[ GetPendingGiftsAsync
	Retrieves all pending gamepass gifts for a player from ProfileStore
	@param userId: The user ID to get gifts for
	@return Promise that resolves with table of pending gifts
]]--
function CommerceHub:GetPendingGiftsAsync(userId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	return Promise.new(function(resolve, reject)
		-- We first check the rate limit for this user
		validateRate(userId, Promise.reject)

		task.spawn(function()
			local success, result = pcall(function()
				local store = MainStore:StartSessionAsync("gifts-" .. userId)
				if not store then
					return {}
				end

				local gifts = store.Data.Gifts or {}
				store:EndSession()
				return gifts
			end)

			if success then
				resolve(result or {})
			else
				reject("Failed to load gifts: " .. tostring(result))
			end
		end)
	end)
end

--[[ ClaimGiftAsync
	Claims a gift and removes it from pending
	@param userId: The user ID claiming the gift
	@param giftIndex: The index of the gift to claim
	@return Promise that resolves when gift is claimed
]]--
function CommerceHub:ClaimGiftAsync(userId, giftIndex)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	if type(giftIndex) ~= "number" or giftIndex < 1 then
		return Promise.reject("Invalid gift index")
	end

	return Promise.new(function(resolve, reject)
		task.spawn(function()
			local success, result = pcall(function()
				local store = MainStore:StartSessionAsync("gifts-" .. userId)
				if not store then
					reject("Failed to load store")
					return
				end

				if not store.Data.Gifts or not store.Data.Gifts[giftIndex] then
					store:EndSession()
					reject("Gift not found")
					return
				end

				local claimedGift = store.Data.Gifts[giftIndex]
				table.remove(store.Data.Gifts, giftIndex)
				store:EndSession()

				resolve(claimedGift)
			end)

			if not success then
				reject("Failed to claim gift: " .. tostring(result))
			end
		end)
	end)
end

--[[ OnGiftReceived
	Returns a signal that fires when a player receives a gift
	@return Signal that fires with gift data
]]--
function CommerceHub:OnGiftReceived()
	return GiftReceivedSignal
end

--[[ ListenToGifts
	Connects a callback to when a player receives a gift
	@param userId: The user ID to listen for gifts
	@param callback: Function(gift) to call when gift received
	@return Connection object with Disconnect method
]]--
function CommerceHub:ListenToGifts(userId, callback)
	if not validateUserId(userId) then
		error("Invalid user ID")
	end

	if type(callback) ~= "function" then
		error("Callback must be a function")
	end

	local connection = GiftReceivedSignal:Connect(function(giftData)
		if giftData.ToUserId == userId then
			callback(giftData)
		end
	end)

	return connection
end

--[[ LoadPlayerGifts
	Loads gifts for a player when they join
	@param userId: The user ID to load gifts for
	@return Promise that resolves when gifts are loaded
]]--
function CommerceHub:LoadPlayerGifts(userId)
	if not validateUserId(userId) then
		return Promise.reject("Invalid user ID")
	end

	return self:GetPendingGiftsAsync(userId):andThen(function(gifts)
		if #gifts > 0 then
			for _, gift in ipairs(gifts) do
				task.defer(function()
					GiftReceivedSignal:Fire({
						ToUserId = userId,
						FromUserId = gift.FromUserId,
						GamepassId = gift.GamepassId,
						Timestamp = gift.Timestamp,
					})
				end)
			end
		end
		return gifts
	end)
end

--// Main
return CommerceHub
```
