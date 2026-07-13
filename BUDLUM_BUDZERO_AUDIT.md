diff --git a/src/core/account.rs b/src/core/account.rs
index 178aaa9..9e06bcd 100644
--- a/src/core/account.rs
+++ b/src/core/account.rs
@@ -471,18 +471,15 @@ impl AccountState {
         if tx.from == Address::zero() {
             return Ok(());
         }
-        if !tx.verify() {
-            return Err("Invalid signature".into());
-        }
-        if tx.fee < self.base_fee {
-            return Err(format!("Fee too low: {} < {}", tx.fee, self.base_fee));
-        }
         if tx.nonce != expected_nonce {
             return Err(format!(
                 "Invalid nonce: expected {}, got {}",
                 expected_nonce, tx.nonce
             ));
         }
+        if tx.fee < self.base_fee {
+            return Err(format!("Fee too low: {} < {}", tx.fee, self.base_fee));
+        }
         let total_cost = tx.total_cost();
         if spendable_balance < total_cost {
             return Err(format!(
@@ -490,6 +487,9 @@ impl AccountState {
                 spendable_balance, total_cost, tx.amount, tx.fee
             ));
         }
+        if !tx.verify() {
+            return Err("Invalid signature".into());
+        }
 
         match tx.tx_type {
             TransactionType::Transfer => {
diff --git a/src/core/transaction.rs b/src/core/transaction.rs
index 66271eb..56d94c7 100644
--- a/src/core/transaction.rs
+++ b/src/core/transaction.rs
@@ -174,8 +174,8 @@ impl Transaction {
     ) -> Self {
         let timestamp = std::time::SystemTime::now()
             .duration_since(std::time::UNIX_EPOCH)
-            .unwrap()
-            .as_millis();
+            .map(|d| d.as_millis())
+            .unwrap_or(0);
         let mut tx = Transaction {
             from,
             to,
diff --git a/src/execution/executor.rs b/src/execution/executor.rs
index fca84be..75b6b0a 100644
--- a/src/execution/executor.rs
+++ b/src/execution/executor.rs
@@ -130,6 +130,21 @@ impl Executor {
                             "Insufficient stake",
                         ));
                     }
+
+                    // Bulgu 3 (Stake Recycling Double Voting Fix):
+                    // Adjust vote totals of active governance proposals if the validator has voted.
+                    for proposal in state.governance.proposals.iter_mut() {
+                        if proposal.status == crate::core::governance::ProposalStatus::Active {
+                            if let Some(&voted_for) = proposal.voters.get(&tx.from) {
+                                if voted_for {
+                                    proposal.votes_for = proposal.votes_for.saturating_sub(tx.amount);
+                                } else {
+                                    proposal.votes_against = proposal.votes_against.saturating_sub(tx.amount);
+                                }
+                            }
+                        }
+                    }
+
                     validator.stake = validator.stake.saturating_sub(tx.amount);
                     if validator.stake == 0 {
                         validator.active = false;
@@ -198,6 +213,16 @@ impl Executor {
                             ));
                         }
                     } else {
+                        // Bulgu 7 (Unrestricted Governance Proposal Creation Fix):
+                        // Ensure that only active validators can create proposals.
+                        let proposer_stake = state.get_validator(&tx.from).map(|v| v.stake).unwrap_or(0);
+                        if proposer_stake == 0 {
+                            return Err(BudlumError::validation(
+                                "governance_proposer_not_validator",
+                                "Only active validators can create proposals",
+                            ));
+                        }
+
                         // Likely a Proposal: [duration (8), ProposalType (...)]
                         let mut dur_bytes = [0u8; 8];
                         dur_bytes.copy_from_slice(&tx.data[0..8]);
diff --git a/src/execution/zkvm.rs b/src/execution/zkvm.rs
index 9e6c645..879dddf 100644
--- a/src/execution/zkvm.rs
+++ b/src/execution/zkvm.rs
@@ -25,7 +25,9 @@ impl ZkVmExecutor {
         }
 
         let program = decode_program(bytecode)?;
-        let mut vm = Vm::with_gas_limit(1024, gas_limit);
+        // Bulgu 14 (Heap Pointer Out-Of-Bounds Fix):
+        // Increased VM memory size from 1024 to 8192 to match the compiler's heap initialization (4096).
+        let mut vm = Vm::with_gas_limit(8192, gas_limit);
 
         let receipt = std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| vm.run(&program)))
             .map_err(|_| "BudZKVM execution failed".to_string())?
@@ -63,7 +65,9 @@ pub fn prove_bytecode(
         return Err("BudZKVM bytecode length must be a multiple of 8 bytes".into());
     }
     let program = decode_program(bytecode)?;
-    let mut vm = Vm::with_gas_limit(1024, gas_limit);
+    // Bulgu 14 (Heap Pointer Out-Of-Bounds Fix):
+    // Increased VM memory size from 1024 to 8192 to match the compiler's heap initialization (4096).
+    let mut vm = Vm::with_gas_limit(8192, gas_limit);
     let receipt = std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| vm.run(&program)))
         .map_err(|_| "BudZKVM execution failed".to_string())?
         .map_err(|_| "BudZKVM execution failed".to_string())?;
diff --git a/src/registry/params.rs b/src/registry/params.rs
index 8187400..8e999fb 100644
--- a/src/registry/params.rs
+++ b/src/registry/params.rs
@@ -78,6 +78,27 @@ impl RegistryParams {
             MaliciousBehaviour => self.malicious_slash_ratio_fixed,
         }
     }
+
+    /// Bulgu 9 (Governance Parameter Bounds Fix):
+    /// Validate governance params to prevent extreme values and system sabotage.
+    pub fn validate(&self) -> Result<(), String> {
+        if self.min_stake < 100 {
+            return Err("min_stake must be at least 100".into());
+        }
+        if self.unbonding_epochs == 0 || self.unbonding_epochs > 100_000 {
+            return Err("unbonding_epochs must be between 1 and 100,000".into());
+        }
+        if self.double_sign_slash_ratio_fixed > FIXED_POINT_SCALE {
+            return Err("double_sign_slash_ratio_fixed cannot exceed FIXED_POINT_SCALE".into());
+        }
+        if self.liveness_slash_ratio_fixed > FIXED_POINT_SCALE {
+            return Err("liveness_slash_ratio_fixed cannot exceed FIXED_POINT_SCALE".into());
+        }
+        if self.malicious_slash_ratio_fixed > FIXED_POINT_SCALE {
+            return Err("malicious_slash_ratio_fixed cannot exceed FIXED_POINT_SCALE".into());
+        }
+        Ok(())
+    }
 }
 
 impl Default for RegistryParams {


=== BUDZERO PATCH ===

diff --git a/bud-state/src/lib.rs b/bud-state/src/lib.rs
index cc7c6ea..6efa60a 100644
--- a/bud-state/src/lib.rs
+++ b/bud-state/src/lib.rs
@@ -26,7 +26,7 @@ pub trait StateBackend {
 pub struct State {
     accounts: HashMap<u64, Account>,
     path: String,
-    backup: Option<HashMap<u64, Account>>,
+    backup_stack: Vec<HashMap<u64, Account>>,
 }
 
 pub fn hash_account(acc: &Account) -> Hash {
@@ -159,12 +159,16 @@ impl State {
         Ok(Self {
             accounts,
             path: path.to_string(),
-            backup: None,
+            backup_stack: Vec::new(),
         })
     }
 
     pub fn save(&self) {
-        self.save_atomic().expect("Failed to save state atomically");
+        // Bulgu 12 (expect panic on state save Fix):
+        // Avoid crashing the entire program if the disk is full or has write permissions issue.
+        if let Err(e) = self.save_atomic() {
+            eprintln!("Error saving state atomically: {}", e);
+        }
     }
 
     pub fn save_to(&self, path: &str) -> Result<(), String> {
@@ -225,16 +229,20 @@ impl StateBackend for State {
     }
 
     fn begin_transaction(&mut self) {
-        self.backup = Some(self.accounts.clone());
+        // Bulgu 5 (State Rollback Stack Fix):
+        // Nested transactions are now safely supported by pushing to a LIFO backup_stack.
+        self.backup_stack.push(self.accounts.clone());
     }
 
     fn commit(&mut self) -> Result<(), String> {
-        self.backup = None;
+        // Pop the committed transaction's backup off the stack.
+        self.backup_stack.pop();
         self.save_atomic()
     }
 
     fn rollback(&mut self) {
-        if let Some(backup) = self.backup.take() {
+        // Restore the parent transaction's unmodified state.
+        if let Some(backup) = self.backup_stack.pop() {
             self.accounts = backup;
         }
     }
