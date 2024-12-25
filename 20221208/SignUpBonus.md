---
marp: true
theme: default
paginate: true
---

# <!--fit--> Technical Review

---

# <!--fit--> Close Sign Up Bonus

---

![bg right](https://i.pinimg.com/originals/f3/d7/d4/f3d7d4221da1bc07593d37c55855bf6f.gif)

# Let's start with John, whose event coming to an end!

---

# So, we have to ...

![bg left](https://i.pinimg.com/originals/f2/42/ba/f242bac4512325947a7284b1afd4d32b.gif)

1. Get John's bonus event data
2. Set his balance to zero
3. Check if he has any running bets
4. Update event status in the database
5. Send event for reset **rollover** and **withdrawable balance**
6. Send event for balance transaction log

---

# And if there's any exception happen ...

![bg right](https://i.pinimg.com/originals/54/1b/8e/541b8e0ba23556d7ba014581e03eb35c.gif)

1. For running bet exception, we rollback his **balance**
2. For other error, we rollback his **balance** and **event status**

---

# <!--fit--> Here comes the complicated part

---

# Base on the turnover and rollover, his event can be ...

### Achieved

Only Rollover is 0 and Turnover has value, then
we skip any steps related to balance.

### Unused

Only turnover is 0 and rollover has value, then
John's balance will be recycled and log in recycle transaction log.

### Failed

Both has value, then
John's balance will be adjusted as normal.

---

# Take a full look

---

###### Service Before

```cs
public async Task<bool> CloseSignUpBonus(CloseSignUpBonusInput input)
{
    var activeEvent = await GetActiveEvent(input);

    var hasRemainRollover = await IsRemainRollOverHasValue(input);
    var setBalanceToZeroResp = new SetBalanceToZeroOutput();
    var balanceToZeroByRecycleOutput = new SetBalanceToZeroByRecycleOutput();
    var hasTurnOver = await HasUsedDuringEventPeriod(activeEvent);

    if (hasRemainRollover)
    {
        if (hasTurnOver)
            setBalanceToZeroResp = await SetBalanceToZero(_mapper.Map<SetBalanceToZeroInput>(activeEvent));
        else
            balanceToZeroByRecycleOutput =
                await SetBalanceToZeroByRecycle(_mapper.Map<SetBalanceToZeroInput>(activeEvent));
    }
```

---

```cs
    var bonusEventEntity = _bonusEventEntityFactory.CreateSignUpEntity(input);
    bonusEventEntity.SetEventEngagement(hasRemainRollover, hasTurnOver);

    try
    {
        await CheckRunningBetsExists(input);
        var proceedTime = setBalanceToZeroResp?.ProceedTime ?? balanceToZeroByRecycleOutput.ProceedTime;
        var affectedRow = await bonusEventEntity.CloseEvent(proceedTime, hasTurnOver, hasRemainRollover);

        if (!hasRemainRollover) return affectedRow > 0;

        await ResetBalanceAndRRO(input);
        switch (hasTurnOver)
        {
            case true:
                if (setBalanceToZeroResp.AdjustAmount != 0m)
                    await _actionLogPublisher.PublishTransactionLog(
                        CreateActionLogTransactionEvent(setBalanceToZeroResp, activeEvent));
                break;
            case false:
                await _bonusPublisher.PublishRecycleBonusEvent(balanceToZeroByRecycleOutput, bonusEventEntity);
                break;
        }

        return affectedRow > 0;
    }
```

---

```cs

    catch (ExistsRunningBetException)
    {
        await HandleError(setBalanceToZeroResp, balanceToZeroByRecycleOutput);
        throw;
    }
    catch (Exception)
    {
        await HandleError(setBalanceToZeroResp, balanceToZeroByRecycleOutput);
        await bonusEventEntity.RecoveryEvent();
        throw;
    }
}


private async Task HandleError(SetBalanceToZeroOutput setBalanceToZeroResp,
    SetBalanceToZeroByRecycleOutput setBalanceToZeroByRecycleResp)
{
    if (setBalanceToZeroResp?.AdjustmentId != null)
        await _depositRequisitionGRpcClient.CancelResetCashBalanceV1(
            CreateRollBackRequest(setBalanceToZeroResp.AdjustmentId.Value, "Bonus Event Has Running Bets"));

    if (setBalanceToZeroByRecycleResp?.TxRecycleId != null)
        await _depositRequisitionGRpcClient.CancelResetCashBalanceByRecycleV1(
            CreateRollBackRequest(setBalanceToZeroByRecycleResp.TxRecycleId.Value,
                "Bonus Event Has Running Bets"));
}

```

---

###### Service After

```cs
public async Task<bool> CloseSignUpBonus(CloseSignUpBonusInput input)
{
    var activeEvent = await GetActiveEvent(input);

    var signUpBonusEntity =
        await _bonusEventEntityFactory.CreateSignUpEntity(activeEvent);

    await signUpBonusEntity.SetBalanceToZero();

    try
    {
        var runningBetCount = await signUpBonusEntity.CheckRunningBets();
        if (runningBetCount > 0)
            throw new ExistsRunningBetException($"RunningBet Exists Count:{runningBetCount}");

        var affectedRow = await signUpBonusEntity.CloseEvent();

        await signUpBonusEntity.ResetBalanceAndRRO();
        await signUpBonusEntity.PublishEvent();
        return affectedRow > 0;
    }
    catch (ExistsRunningBetException)
    {
        await signUpBonusEntity.CancelSetBalanceToZero();
        throw;
    }
    catch (Exception)
    {
        await signUpBonusEntity.CancelSetBalanceToZero();
        await signUpBonusEntity.RecoveryEvent();
        throw;
    }
}
```

---

###### Extract Player Service After

```cs
public async Task<EngagementType> GetEngagementType(string playerId, DateTime claimedOn)
{
    var hasRemainRollover = await GetHasRemainRollover(playerId);
    var hasTurnOver = await GetHasTurnOver(playerId, claimedOn);
    if (!hasRemainRollover && !hasTurnOver) throw new Exception();

    return hasRemainRollover
        ? hasTurnOver ? EngagementType.Failed : EngagementType.Unused
        : EngagementType.Achieved;
}

```

---

###### Factory Before

```cs
public IBonusEventEntity CreateSignUpEntity(CloseSignUpBonusInput closeInput)
{
    var bonusEventData = _mapper.Map<PlayerBonusEventData>(closeInput);

    bonusEventData.TransactionId = NewTransactionId();
    bonusEventData.ClosedOn = closeInput.ClosedOn ?? DateTime.UtcNow;
    bonusEventData.EventName = EventType.SignUpBonus;

    return new BonusEventEntity(
        _bonusEventRepository,
        _eventSettingRepository,
        _unitOfWork,
        _mapper)
    {
        EventName = EventType.SignUpBonus,
        Data = bonusEventData
    };
}
```

---

###### Factory After

```cs
public async Task<CloseSignUpBonusEventEntity> CreateSignUpEntity(
    BonusRecordInfo activeEvent)
{
    activeEvent.TransactionId = NewTransactionId();
    if (!activeEvent.ClaimedOn.HasValue) throw new Exception("Bonus Record claimed on null");
    activeEvent.Engagement =
        await _bonusPlayerService.GetEngagementType(activeEvent.PlayerId, activeEvent.ClaimedOn.Value);

    switch (activeEvent.Engagement)
    {
        case EngagementType.Achieved:
            return new CloseAchievedSignUpBonusEventEntity(_rolloverReaderGRpcClient, _betReaderGRpcClient,
                _bonusEventRepository, _unitOfWork, _mapper) { Data = activeEvent };
        case EngagementType.Failed:
            return new CloseFailedSignUpBonusEventEntity(_rolloverReaderGRpcClient, _betReaderGRpcClient,
                _bonusEventRepository, _unitOfWork, _mapper, _depositRequisitionGRpcClient, _actionLogPublisher)
            {
                Data = activeEvent
            };
        case EngagementType.Unused:
            return new CloseUnusedSignUpBonusEventEntity(_rolloverReaderGRpcClient, _betReaderGRpcClient,
                _bonusEventRepository, _unitOfWork, _mapper, _depositRequisitionGRpcClient, _bonusPublisher)
            {
                Data = activeEvent
            };
        case EngagementType.Initial:
        default:
            throw new Exception("Abnormal");
    }
}

```

---

###### Entity Before

```cs
namespace BlueWhale.Bonus.Event.Domain.Entities
{
    public class SignUpBonusEntity : ISignUpBonusEntity
    {
        private readonly IBonusEventRepository _bonusEventRepository;
        private readonly IBonusEventStatusUnitOfWork _unitOfWork;
        private readonly IEventSettingRepository _eventSettingRepository;
        private EventRolloverSetting _eventSettingRolloverSetting;
        private readonly IMapper _mapper;
        public PlayerBonusEventData Data { get; set; }
        public EventType EventName { get; set; }

        public SignUpBonusEntity(
            IBonusEventRepository bonusEventRepository,
            IEventSettingRepository eventSettingRepository,
            IBonusEventStatusUnitOfWork unitOfWork,
            IMapper mapper)
        {
            _bonusEventRepository = bonusEventRepository;
            _eventSettingRepository = eventSettingRepository;
            _unitOfWork = unitOfWork;
            _mapper = mapper;
        }

        public async Task SaveEvent(DateTime transactionTime){}

        public BonusMessage CreateBonusEvent()
        {
            return new BonusMessage()
            {
                PlayerId = Data.PlayerId,
                TransactionTime = Data.RegisterDate.Value,
                Amount = (decimal)Data.Amount.ToRealBalance(),
                TransactionId = Data.TransactionId,
            };
        }

        public BonusMessage CreateRecycleBonusEvent(SetBalanceToZeroByRecycleOutput output)
        {
            return new BonusMessage()
            {
                PlayerId = Data.PlayerId,
                TransactionTime = output.ProceedTime.Value,
                Amount = output.RecycleAmount,
                TransactionId = output.TxRecycleId.ToString(),
            };
        }

        public RolloverTransactionEvent CreateRolloverEvent(CashWalletOutput cashWalletOutput)
        {
            return new RolloverTransactionEvent()
            {
                DepositTimes = Data.DepositTimes,
                PlayerId = Data.PlayerId,
                EventTime = Data.RegisterDate.Value,
                Balance = cashWalletOutput.Balance,
                Amount = Data.Amount,
                TransactionId = Data.TransactionId,
                RolloverTransactionType = (int)RolloverTransactionType.Deposit
            };
        }

        public void UpdateEventInfo(){}

        public void CheckEventAvailable(ClaimSignUpBonusInput input, string playerType){}

        public async Task CheckDuplicateClaim(){}

        public async Task<int> CloseEvent(DateTime? proceedTime)
        {
            Data.EventStatus = StatusType.Closed;
            if (proceedTime.HasValue)
            {
                Data.ClosedOn = proceedTime;
            }
            var affectedRow = await _unitOfWork.UpdateStatus(_mapper.Map<UpdateBonusStatusInput>(Data));
            await _bonusEventRepository.DeleteCache(Data.PlayerId);
            return affectedRow;
        }

        public async Task RecoveryEvent()
        {
            Data.EventStatus = StatusType.Claimed;
            Data.Engagement = EngagementType.Initial;
            await _unitOfWork.UpdateStatus(_mapper.Map<UpdateBonusStatusInput>(Data));
        }

        private void GetEventToggle(EventType eventName){}

        private EventRolloverSetting GetEventSetting(EventType eventName){}
    }
}

```

---

###### Entity After

```cs
public abstract class CloseSignUpBonusEventEntity
{
    public BonusRecordInfo Data { get; set; }
    public AdjustBalanceOutput AdjustBalanceResult { get; set; }
    protected IRolloverReaderGRpcClient RolloverReaderGRpcClient;
    protected IBetReaderGRpcClient BetReaderGRpcClient;
    protected IBonusEventRepository BonusEventRepository;
    protected IBonusEventStatusUnitOfWork UnitOfWork;
    protected IMapper Mapper;

    public abstract Task SetBalanceToZero();

    public virtual async Task<int> CheckRunningBets()
    {
        var resp = await BetReaderGRpcClient.SearchRunningBetsV1(new SearchRunningBetsRequest
        {
            StartTime = DateTime.UtcNow.AddDays(-3).ToTick(),
            EndTime = DateTime.UtcNow.ToTick(),
            PlayerId = Data.PlayerId,
            CurrentPage = 1,
            PageSize = 1
        });
        return resp.RowCount;
    }

    public virtual async Task<int> CloseEvent()
    {
        Data.IsClosed = true;
        var affectedRow = await UnitOfWork.UpdateStatus(Mapper.Map<UpdateBonusStatusInput>(Data));
        await BonusEventRepository.DeleteCache(Data.PlayerId);
        return affectedRow;
    }

    public virtual async Task ResetBalanceAndRRO()
    {
        await RolloverReaderGRpcClient.ResetBalanceAndRolloverV1(new ResetBalanceAndRolloverRequest()
            { PlayerId = Data.PlayerId });
    }

    public abstract Task PublishEvent();

    public abstract Task CancelSetBalanceToZero();

    public virtual async Task RecoveryEvent()
    {
        Data.Status = StatusType.Claimed;
        Data.IsClosed = false;
        Data.Engagement = EngagementType.Initial;
        Data.ClosedOn = null;
        await UnitOfWork.UpdateStatus(Mapper.Map<UpdateBonusStatusInput>(Data));
    }
}
```

---

###### Achieved After

```cs
public class CloseAchievedSignUpBonusEventEntity : CloseSignUpBonusEventEntity
{
    public CloseAchievedSignUpBonusEventEntity(
        IRolloverReaderGRpcClient rolloverReaderGRpcClient,
        IBetReaderGRpcClient betReaderGRpcClient,
        IBonusEventRepository bonusEventRepository,
        IBonusEventStatusUnitOfWork unitOfWork,
        IMapper mapper)
    {
        RolloverReaderGRpcClient = rolloverReaderGRpcClient;
        BetReaderGRpcClient = betReaderGRpcClient;
        BonusEventRepository = bonusEventRepository;
        UnitOfWork = unitOfWork;
        Mapper = mapper;
    }


    public override Task SetBalanceToZero()
    {
        return Task.CompletedTask;
    }

    public override Task ResetBalanceAndRRO()
    {
        return Task.CompletedTask;
    }

    public override Task PublishEvent()
    {
        return Task.CompletedTask;
    }

    public override Task CancelSetBalanceToZero()
    {
        return Task.CompletedTask;
    }
}
```

---

###### Unused After

```cs
public class CloseUnusedSignUpBonusEventEntity : CloseSignUpBonusEventEntity
{
    private readonly ICashCounterDepositRequisitionGRpcClient _depositRequisitionGRpcClient;
    private readonly IBonusPublisher _bonusPublisher;

    public CloseUnusedSignUpBonusEventEntity(
        IRolloverReaderGRpcClient rolloverReaderGRpcClient,
        IBetReaderGRpcClient betReaderGRpcClient,
        IBonusEventRepository bonusEventRepository,
        IBonusEventStatusUnitOfWork unitOfWork, IMapper mapper,
        ICashCounterDepositRequisitionGRpcClient depositRequisitionGRpcClient,
        IBonusPublisher bonusPublisher)
    {
        _depositRequisitionGRpcClient = depositRequisitionGRpcClient;
        RolloverReaderGRpcClient = rolloverReaderGRpcClient;
        _bonusPublisher = bonusPublisher;
        Mapper = mapper;
        BetReaderGRpcClient = betReaderGRpcClient;
        BonusEventRepository = bonusEventRepository;
        UnitOfWork = unitOfWork;
    }


    public override async Task SetBalanceToZero()
    {
        var resp = await _depositRequisitionGRpcClient.ResetCashBalanceByRecycleV1(
            new ResetCashBalanceRequest()
            {
                Currency = Data.Currency,
                PlayerId = Data.PlayerId,
                RequestIp = "127.0.0.1",
                TriggeredBy = "Bonus.Event"
            });
        AdjustBalanceResult = Mapper.Map<AdjustBalanceOutput>(resp);
        Data.ClosedOn = AdjustBalanceResult.ProceedTime;
    }

    public override async Task PublishEvent()
    {
        await _bonusPublisher.PublishTransactionEvent(new BonusMessage
        {
            PlayerId = Data.PlayerId,
            TransactionTime = AdjustBalanceResult.ProceedTime,
            Amount = AdjustBalanceResult.Amount,
            TransactionId = AdjustBalanceResult.Id.ToString(),
        });
    }

    public override async Task CancelSetBalanceToZero()
    {
        await _depositRequisitionGRpcClient.CancelResetCashBalanceByRecycleV1(
            new AuditRequisitionRequest
            {
                Reason = "Bonus Event Has Running Bets", RequisitionId = AdjustBalanceResult.Id.ToString(),
                TriggeredBy = "Bonus.Event", RequestIp = "127.0.0.1"
            });
    }
}
```

---

###### Failed After

```cs
public class CloseFailedSignUpBonusEventEntity : CloseSignUpBonusEventEntity
{
    private readonly ICashCounterDepositRequisitionGRpcClient _depositRequisitionGRpcClient;
    private readonly IActionLogPublisher _actionLogPublisher;

    public CloseFailedSignUpBonusEventEntity(
        IRolloverReaderGRpcClient rolloverReaderGRpcClient,
        IBetReaderGRpcClient betReaderGRpcClient,
        IBonusEventRepository bonusEventRepository,
        IBonusEventStatusUnitOfWork unitOfWork,
        IMapper mapper,
        ICashCounterDepositRequisitionGRpcClient depositRequisitionGRpcClient,
        IActionLogPublisher actionLogPublisher
    )
    {
        _depositRequisitionGRpcClient = depositRequisitionGRpcClient;
        RolloverReaderGRpcClient = rolloverReaderGRpcClient;
        BetReaderGRpcClient = betReaderGRpcClient;
        _actionLogPublisher = actionLogPublisher;
        BonusEventRepository = bonusEventRepository;
        UnitOfWork = unitOfWork;
        Mapper = mapper;
    }


    public override async Task SetBalanceToZero()
    {
        var resp = await _depositRequisitionGRpcClient.ResetCashBalanceV1(
            new ResetCashBalanceRequest
            {
                Currency = Data.Currency,
                PlayerId = Data.PlayerId,
                RequestIp = "127.0.0.1",
                TriggeredBy = "Bonus.Event"
            });
        AdjustBalanceResult = Mapper.Map<AdjustBalanceOutput>(resp);
        Data.ClosedOn = AdjustBalanceResult.ProceedTime;
    }

    public override async Task PublishEvent()
    {
        if (AdjustBalanceResult.Amount != 0m)
        {
            var transactionEvent = new ActionLogTransactionEvent
            {
                AccountId = Data.PlayerId,
                ActionBy = "System",
                Action = "Transaction",
                ActionIp = "127.0.0.1",
                ActionTime = Data.ClosedOn ?? DateTime.Now,
                AfterBalance = 0m,
                Amount = AdjustBalanceResult.Amount,
                LogTransactionId = AdjustBalanceResult.Id.ToString(),
                TransactionType = "Adjustment",
                ApprovedBy = "Bonus.Event",
                Currency = Data.Currency
            };
            await _actionLogPublisher.PublishTransactionLog(transactionEvent);
        }
    }

    public override async Task CancelSetBalanceToZero()
    {
        await _depositRequisitionGRpcClient.CancelResetCashBalanceV1(
            new AuditRequisitionRequest
            {
                Reason = "Bonus Event Has Running Bets", RequisitionId = AdjustBalanceResult.Id.ToString(),
                TriggeredBy = "Bonus.Event", RequestIp = "127.0.0.1"
            });
    }
}
```

---

###### Action Log Publisher Before

```cs

 public class ActionLogPublisher : IActionLogPublisher
    {
        private readonly Identifier _identifier;
        private readonly IMessagePublisher _messagePublisher;

        public ActionLogPublisher(IMessagePublisher messagePublisher,
            Identifier identifier)
        {
            _messagePublisher = messagePublisher;
            _identifier = identifier;
        }

        public Task PublishTransactionLog(ActionLogTransactionEvent transactionEvent)
        {
            transactionEvent.RequestId = _identifier.IdentifyKey;
            Task.Factory.StartNew(async () =>
            {
                await _messagePublisher.PublishAsync("bw_bo_actionlog_cmd", transactionEvent.AccountId,
                    transactionEvent);
            });

            return Task.CompletedTask;
        }

        public Task PublishAdjustmentLog(BonusEventEntity entity, SetBalanceToZeroOutput txOutput)
        {
            PublishTransactionLog(CreateAdjustmentEvent(txOutput, entity));
            return Task.CompletedTask;
        }
        private static ActionLogTransactionEvent CreateAdjustmentEvent(SetBalanceToZeroOutput tx,
            BonusEventEntity entity)
        {
            return new ActionLogTransactionEvent
            {
                AccountId = entity.Data.PlayerId,
                ActionBy = "System",
                Action = "Transaction",
                ActionIp = "0.0.0.0",
                ActionTime = entity.Data.ClosedOn.GetValueOrDefault(DateTime.UtcNow),
                AfterBalance = 0,
                Amount = tx.AdjustAmount,
                LogTransactionId = tx.AdjustmentId.ToString(),
                TransactionType = "Adjustment",
                ApprovedBy = "Bonus.Event",
                Currency = entity.Data.Currency
            };
        }
    }

```

---
