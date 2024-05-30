1. Wejdź do es_extended > server > main.lua

2. Dodaj w zmiennej loadPlayer `account_number` w taki sposób
local loadPlayer = 'SELECT `accounts`, `job`, `job_grade`, `group`, `position`, `inventory`, `skin`, `loadout`, `metadata`, `account_number`'

3. Następnie w newPlayer dodaj `account_number` = ? w taki sposób
if Config.Multichar then
	newPlayer = newPlayer .. ', `firstname` = ?, `lastname` = ?, `dateofbirth` = ?, `sex` = ?, `nationality` = ?, `account_number` = ?'
end

4. Następnie wyszukaj funkcję loadESXPlayer i w userData dodaj accNumber w taki sposób
local userData = {
	accounts = {},
	inventory = {},
	job = {},
	loadout = {},
	playerName = GetPlayerName(playerId),
	weight = 0,
	metadata = {},
	accNumber = nil,
}

5. Następnie pod local result dodaj userData.accNumber = result.account_number

6. Następnie zjedź niżej do local xPlayer = CreateExtendedPlayer i na końcu w argumencie dodaj userData.accNumber w taki sposób
local xPlayer = CreateExtendedPlayer(playerId, identifier, userData.group, userData.accounts, userData.inventory, userData.weight, userData.job, userData.loadout, userData.playerName, userData.coords, userData.metadata, userData.accNumber)

7. Potem lekko niżej mamy if userData.firstname then i pod tym dodajemy
if userData.accNumber then
	xPlayer.set('accNumber', userData.accNumber)
end

8. Na koniec wyszukaj funkcję createESXPlayer i dodaj exports['piotreq_banking']:GenerateNumber() na koncu tabeli w tym miejscu
MySQL.prepare(newPlayer, 
{ json.encode(accounts), identifier, defaultGroup, data.firstname, data.lastname, data.dateofbirth, data.sex, data.nationality, exports['piotreq_banking']:GenerateNumber() }, function()
	loadESXPlayer(identifier, playerId, true)
end)

9. Wejdź do Wejdź do es_extended > server > classes > player.lua oraz w funkcji CreateExtendedPlayer dopisz argument accNumber w taki sposob
function CreateExtendedPlayer(playerId, identifier, group, accounts, inventory, weight, job, loadout, name, coords, metadata, accNumber)

10. Następnie pod local self = {} dopisz self.accNumber = accNumber

11. Wejdź do es_extended > server > classes > overrides > oxinventory.lua

12. Znajdź funkcję addAccountMoney oraz removeAccountMoney i zamień je na to

	addAccountMoney = function(self)
		return function(accountName, money, reason, sender, name)
			reason = reason or 'Wpłata na konto'
			sender = sender or 'SYSTEM'
			name = name or 'SYSTEM'
			if money > 0 then
				local account = self.getAccount(accountName)

				if account then
					money = account.round and ESX.Math.Round(money) or money
					self.accounts[account.index].money = self.accounts[account.index].money + money
					self.triggerEvent('esx:setAccountMoney', account)
					TriggerEvent('esx:addAccountMoney', self.source, accountName, money, reason)
					if (reason ~= "Minutówka") then
						if accountName == 'bank' then
							exports['piotreq_banking']:AddHistory(sender, self.accNumber, 1, money, name, reason)
							exports['piotreq_banking']:UpdateAccount(self.accNumber, 'add', money)
						end
					end
					if Inventory.accounts[accountName] then
						Inventory.AddItem(self.source, accountName, money)
					end
				end
			end
		end
	end,

	removeAccountMoney = function(self)
		return function(accountName, money, reason, sender, name)
			reason = reason or 'Wypłata z konta'
			sender = sender or 'SYSTEM'
			name = name or 'SYSTEM'
			if money > 0 then
				local account = self.getAccount(accountName)

				if account then
					money = account.round and ESX.Math.Round(money) or money
					self.accounts[account.index].money = self.accounts[account.index].money - money
					self.triggerEvent('esx:setAccountMoney', account)
					TriggerEvent('esx:removeAccountMoney', self.source, accountName, money, reason)
					if accountName == 'bank' then
						exports['piotreq_banking']:AddHistory(sender, self.accNumber, 2, money, name, reason)
						exports['piotreq_banking']:UpdateAccount(self.accNumber, 'remove', money)
					end
					if Inventory.accounts[accountName] then
						Inventory.RemoveItem(self.source, accountName, money)
					end
				end
			end
		end
	end,

13. Następnie wejdź do esx_addonaccount > server > classes > addonaccount.lua

14. W funkcji CreateAddonAccount dodaj argument number w taki sposób
function CreateAddonAccount(name, owner, money, number, label)

15. Następnie dodaj self.number = number oraz self.label = label w funkcji

16. Znajdź funkcję self.addMoney oraz self.removeMoney i zamień je na to

	function self.addMoney(m, t, s, name)
		s = s or 'SYSTEM'
		t = t or 'Wpłata na konto'
		name = name or 'SYSTEM'
		exports['piotreq_banking']:AddHistory(s, self.number, 1, m, name, t)
		exports['piotreq_banking']:UpdateAccount(self.number, 'add', m)
		self.money = self.money + m
		self.save()
	end

	function self.removeMoney(m, t, s, name)
		s = s or 'SYSTEM'
		t = t or 'Wypłata z konta'
		name = name or 'SYSTEM'
		exports['piotreq_banking']:AddHistory(s, self.number, 2, m, name, t)
		exports['piotreq_banking']:UpdateAccount(self.number, 'remove', m)
		self.money = self.money - m
		self.save()
	end

17. Przejdź do server > main.lua linijka 17-21 zeby bylo tak
	if account.money and account.number then
		SharedAccounts[account.name] = CreateAddonAccount(account.name, nil, account.money, account.number, account.label)
	else
		newAccounts[#newAccounts + 1] = {account.name, 0}
	end

18. Następnie linijka 30 ma wyglądać w taki sposób
SharedAccounts[newAccount[1]] = CreateAddonAccount(newAccount[1], nil, 0, exports['piotreq_banking']:GenerateSocietyNumber())

19. Dodaj kolumne w addon_account o nazwie number int(11) oraz label varchar(30)

20. Jeśli używamy eventu esx_addonaccount:refreshAccounts na samym dole możemy też dodać aby zczytywało numer konta dla society
