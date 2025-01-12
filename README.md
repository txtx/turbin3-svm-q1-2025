# turbin3-svm-q1-2025

### Which area are you focusing on for the upcoming cohort? 
SVM-API / PoH-Turbine

### Based on your review of SolanaHowitWorks and any other documentation, describe what is happening in the area you plan to focus on- this can be something very specific- please avoid being too high level.
SolanaHowitWorks is a great resource, giving a good overview of Solana internals in less than 50 slides.
Based on the work I want to focus on, I believe the TPU & TVU are the parts I'd like to focus on.

### Go to the Anza Agave Client Repository - copy 20-30 lines of code from the area you are focusing on.
```rust
    pub fn load_and_execute_sanitized_transactions<CB: TransactionProcessingCallback>(
        &self,
        callbacks: &CB,
        sanitized_txs: &[impl SVMTransaction],
        check_results: Vec<TransactionCheckResult>,
        environment: &TransactionProcessingEnvironment,
        config: &TransactionProcessingConfig,
    ) -> LoadAndExecuteSanitizedTransactionsOutput {

        // [...]

        let (validation_results, validate_fees_us) = measure_us!(self.validate_fees(
            callbacks,
            config.account_overrides,
            sanitized_txs,
            check_results,
            &environment.feature_set,
            environment
                .fee_structure
                .unwrap_or(&FeeStructure::default()),
            environment
                .rent_collector
                .unwrap_or(&RentCollector::default()),
            &mut error_metrics
        ));

        let (mut program_cache_for_tx_batch, program_cache_us) = measure_us!({
            let mut program_accounts_map = Self::filter_executable_program_accounts(
                callbacks,
                sanitized_txs,
                &validation_results,
                PROGRAM_OWNERS,
            );
            for builtin_program in self.builtin_program_ids.read().unwrap().iter() {
                program_accounts_map.insert(*builtin_program, 0);
            }

            let program_cache_for_tx_batch = self.replenish_program_cache(
                callbacks,
                &program_accounts_map,
                config.check_program_modification_slot,
                config.limit_to_load_programs,
            );

            if program_cache_for_tx_batch.hit_max_limit {
                return LoadAndExecuteSanitizedTransactionsOutput {
                    error_metrics,
                    execute_timings,
                    processing_results: (0..sanitized_txs.len())
                        .map(|_| Err(TransactionError::ProgramCacheHitMaxLimit))
                        .collect(),
                };
            }

            program_cache_for_tx_batch
        });

```
This method seems to be main entrypoint to the SVM.
In the call to `validate_fees`, we're making sure that we'll be able to debit fees from the balance of the account paying for the transaction fees.
We are wrapping this call with the macro `measure_us` - measuring the execution time in micro seconds.

Now that we know that the fees can be paid, with the next part we start by loading the executable accounts (programs owned by any loader), along with the builtin programs. The documentation seems sparse, but I believe builtin programs are another denomination for "pre-compiles" (see [SIMD 152](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0152-precompiles.md)).

The missing entries from this map of programs are inserted into a cache, which will most likely help speeding up the execution.

Possible optimization:

```rust
let program_to_store = program_to_load.map(|(key, count)| {
    // Load, verify and compile one program.
    let program = load_program_with_pubkey(
        callback,
        &program_cache.get_environments_for_epoch(self.epoch),
        &key,
        self.slot,
        false,
    )
    .expect("called load_program_with_pubkey() with nonexistent account");
    program.tx_usage_counter.store(count, Ordering::Relaxed);
    (key, program)
});
```
The code is involving an abstraction here (`TransactionProcessingCallback`) so I could be wrong, but in the function `replenish_program_cache`, we are looping over the list of missing programs not present in the cache and fetching them from storage (disk?), one by one. I am not familiar with the storage engine used in Solana, but if range requests is a feature available, I wonder if they could help optimize I/O.


### Are you familiar with various benchmark testing practices? If so, what have you done? If not, do some research and note what you want to learn more about. 
I have been working with `bench` is the past, to measure the cost of the native functions available in the Clarity language (Stacks blockchain). 
These benchmarks have then be used for attributing some costs, helping determining transactions fees. 


### Are you currently working on a project for which you plan to use what you are doing in this program.
I am working on `https://github.com/txtx`, and a better understanding of Solana would be incredibly useful, and would help me design relevant features.
