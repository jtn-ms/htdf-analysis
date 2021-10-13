### Flow(v0->v1)
```
[v0]
BaseApp:EndBlock---------->app.Engine.GetCurrentProtocol().GetEndBlocker()(app.deliverState.ctx, req)
Protocol_0.EndBlocker----->upgrade:EndBlocker
```
### AppVersion vs. ProtocolVersion
***x/upgrade/handler.go:EndBlocker***
```go
func EndBlocker(ctx sdk.Context, uk Keeper) {

	ctx = ctx.WithLogger(ctx.Logger().With("handler", "endBlock").With("module", "htdf/upgrade")).WithEventManager(sdk.NewEventManager())

	upgradeConfig, ok := uk.protocolKeeper.GetUpgradeConfig(ctx)
	if ok {

		versionIDstr := strconv.FormatUint(upgradeConfig.Protocol.Version, 10)
		uk.metrics.Upgrade.Set(float64(upgradeConfig.Protocol.Version))

		validator, found := uk.sk.GetValidatorByConsAddr(ctx, (sdk.ConsAddress)(ctx.BlockHeader().ProposerAddress))
		if !found {
			panic(fmt.Sprintf("validator with consensus-address %s not found", (sdk.ConsAddress)(ctx.BlockHeader().ProposerAddress).String()))
		}

		if ctx.BlockHeader().Version.App == upgradeConfig.Protocol.Version {
			uk.SetSignal(ctx, upgradeConfig.Protocol.Version, validator.ConsAddress().String())
			uk.metrics.Signal.With(ValidatorLabel, validator.ConsAddress().String(), VersionLabel, versionIDstr).Set(1)

			ctx.Logger().Info("Validator has downloaded the latest software ",
				"validator", validator.GetOperator().String(), "version", upgradeConfig.Protocol.Version)

		} else {

			ok := uk.DeleteSignal(ctx, upgradeConfig.Protocol.Version, validator.ConsAddress().String())
			uk.metrics.Signal.With(ValidatorLabel, validator.ConsAddress().String(), VersionLabel, versionIDstr).Set(0)

			if ok {
				ctx.Logger().Info("Validator has restarted the old software ",
					"validator", validator.GetOperator().String(), "version", upgradeConfig.Protocol.Version)
			}
		}

		if uint64(ctx.BlockHeight())+1 == upgradeConfig.Protocol.Height {
			success := tally(ctx, upgradeConfig.Protocol.Version, uk, upgradeConfig.Protocol.Threshold)

			if success {
				ctx.Logger().Info("Software Upgrade is successful.", "version", upgradeConfig.Protocol.Version)
				uk.protocolKeeper.SetCurrentVersion(ctx, upgradeConfig.Protocol.Version)
			} else {
				ctx.Logger().Info("Software Upgrade is failure.", "version", upgradeConfig.Protocol.Version)
				uk.protocolKeeper.SetLastFailedVersion(ctx, upgradeConfig.Protocol.Version)
			}

			uk.AddNewVersionInfo(ctx, NewVersionInfo(upgradeConfig, success))
			uk.protocolKeeper.ClearUpgradeConfig(ctx)
		}
	} else {
		uk.metrics.Upgrade.Set(float64(0))
	}

	ctx.EventManager().EmitEvent(
		sdk.NewEvent(
			"upgrade",
			sdk.NewAttribute(sdk.AppVersionTag, strconv.FormatUint(uk.protocolKeeper.GetCurrentVersion(ctx), 10)),
		),
	)

}
```
***app/baseapp.go:EndBlock***
```go
// EndBlock implements the ABCI interface.
func (app *BaseApp) EndBlock(req abci.RequestEndBlock) (res abci.ResponseEndBlock) {
	app.logger.Info("=======*****============EndBlock=========**********===============")
	if app.deliverState.ms.TracingEnabled() {
		app.deliverState.ms = app.deliverState.ms.SetTracingContext(nil).(sdk.CacheMultiStore)
	}

	// if app.endBlocker != nil {
	// 	res = app.endBlocker(app.deliverState.ctx, req)
	// }
	fmt.Println("app.Engine.GetCurrentProtocol().GetVersion():", app.Engine.GetCurrentProtocol().GetVersion())
	endBlocker := app.Engine.GetCurrentProtocol().GetEndBlocker()
	if endBlocker != nil {
		res = endBlocker(app.deliverState.ctx, req)
	}
	_, appVersionStr, ok := abci.GetEventByKey(res.Events, sdk.AppVersionTag)
	if !ok {
		app.logger.Info("abci.GetEventByKey failed...........")
		return
	}

	appVersion, _ := strconv.ParseUint(string(appVersionStr.GetValue()), 10, 64)
	if appVersion <= app.Engine.GetCurrentVersion() {
		app.logger.Info("appVersion <= app.Engine.GetCurrentVersion..")
		return
	}

	fmt.Print("111111111111	", appVersion, "	22222222222	", app.Engine.GetCurrentVersion(), "\n")
	app.logger.Info(fmt.Sprintf("=== Upgrading protocol from current version: (%v) to version: (%v) ===", app.Engine.GetCurrentVersion(), appVersion))

	success := app.Engine.Activate(appVersion)
	if success {
		app.txDecoder = auth.DefaultTxDecoder(app.Engine.GetCurrentProtocol().GetCodec())
		return
	}

	if upgradeConfig, ok := app.Engine.ProtocolKeeper.GetUpgradeConfigByStore(app.GetKVStore(protocol.KeyMain)); ok {
		res.Events = append(res.Events,
			sdk.MakeEvent("upgrade", "UpgradeFailureTagKey",
				("Please install the right application version from "+upgradeConfig.Protocol.Software)))
	} else {
		res.Events = append(res.Events,
			sdk.MakeEvent("upgrade", "UpgradeFailureTagKey", ("Please install the right application version")))
	}

	return
}
```
***app/vx/protocol_x.go:EndBlocker***
```go
// application updates every end block
func (p *ProtocolV1) EndBlocker(ctx sdk.Context, req abci.RequestEndBlock) abci.ResponseEndBlock {

	//2021-05-12, yqq
	// we should keep all Events in EventManager and return to Tendermint
	ctx = ctx.WithEventManager(sdk.NewEventManager())

	gov.EndBlocker(ctx, p.govKeeper)
	// slashing.EndBlocker(ctx, req, p.slashingKeeper)
	service.EndBlocker(ctx, p.serviceKeeper)
	upgrade.EndBlocker(ctx, p.upgradeKeeper)
	validatorUpdates := stake.EndBlocker(ctx, p.StakeKeeper)

	if p.invCheckPeriod != 0 && ctx.BlockHeight()%int64(p.invCheckPeriod) == 0 {
		p.assertRuntimeInvariants(ctx)
	}

	evs := ctx.EventManager().ABCIEvents()
	// logrus.Infof("%v", evs)

	return abci.ResponseEndBlock{
		ValidatorUpdates: validatorUpdates,
		// Tags:             tags,
		Events: evs, //ctx.EventManager().ABCIEvents(),
	}
}
```
