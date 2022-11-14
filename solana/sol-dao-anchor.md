# Introduction

Suppose that we have a Tournament program deployed(we do not actually!), managing on-chain tournaments & payments. Tournaments can be joined by individual players and teams. Every team has a team captain. When a team wins a prize, it is distributed to the team captain. Team captain is tasked with distributing the prize to the players from his team (in a fair manner, and as agreed inside the team) outside the platform. We would like to bring this aspect of team winnings & prize distribution under the umbrella of features.

### Program Features

Our Task in this tutorial: Create a DAO Voting program for guilds/teams management. The program should offer the functionality to:

1. Create a new team
2. Captain can add or remove members to team
3. People can leave the team
4. Create a YES or NO voting and vote on whether the team will join the tournament X. Team captain would create the voting and all players must vote. Voting is won by majority. Team size fixed to 5.
5. Create a YES or NO voting on how a team will distribute a prize for tournament X. Team captain would create the voting and all players must vote. Voting is won by 2/3 majority. For example, the team captain can propose a voting for example, if there are 5 players the team captain can create voting about prize distribution such as [20%, 20%, 20%, 20%, 20%], [25%, 25%, 20%, 15%, 15%], [%96, %1, %1, %1, %1] and members will vote YES or NO to proposal.
6. People can claim their prize. Tournament program would send the winning prize to the DAO program, specifying the tournament and the team.
7. Transfer the team ownership to another member.

#### Notes: 

The TournamentDAO program does not exist on Solana. Participants do not need to know about it and its functionalities except that it manages tournaments.

# Prerequisites

- Intermediate understanding of Rust, Solana, Anchor for program and JS/TS for testing.
- Basic knowledge of PDA and Accounts in Solana

# Requirements

 <ul>
    <li>Rust installation -> <a href="https://www.rust-lang.org/tools/install">here</a></li>
    <li>Solana installation -> <a href="https://docs.solana.com/cli/install-solana-cli-tools">here</a></li>
    <li>Yarn installation -> <a href="https://yarnpkg.com/getting-started/install">here</a></li>
    <li>Anchor installation -> <a href="https://www.anchor-lang.com/docs/installation">here</a>
    <li>Git installation -> <a href="https://git-scm.com/book/en/v2/Getting-Started-Installing-Git">here</a>
  </ul>
  
 
## Init an anchor project
``` anchor init TeamDAO ```

<b>NOTE: </b>After the initialization is completed. We will write our program and every code in the lib.rs file. 

## Write Create Team Functionality

First we will create Team account struct:

```rust
#[account]
pub struct TeamAccount {
    pub captain: Pubkey,
    pub bump: u8,
    pub name: String,
    pub members: Vec<Pubkey>,
    pub id: u64,
}
```

After creating Team account we will implement create_team function:

```rust
#[program]
pub mod team_dao {
    use super::*;

    pub fn create_team(ctx: Context<CreateTeam>, team_name: String, team_id: u64) -> Result<()> {
        let team = &mut ctx.accounts.team_account;

        team.bump = *ctx
            .bumps
            .get("team_account")
            .ok_or(ErrorCode::InvalidBumpSeeds)?;

        // assigning required parameters to the team
        team.name = team_name;
        team.captain = *ctx.accounts.signer.key;
        team.id = team_id;
        team.members.push(*ctx.accounts.signer.key);
        team.can_join_tournament = false;
        team.distribution_voting_result = false;

        msg!("Team created");
        msg!("Team name: {}", team.name);
        msg!("Team captain: {}", team.captain);

        Ok(())
    }
}
```


After creating create_team function we will handle instructions of create team functionality which will be passed as ctx parameters of create_team function:

```rust
#[derive(Accounts)]
#[instruction(_team_name: String, _team_id: u64)]
pub struct CreateTeam<'info> {
    #[account(init, payer = signer, space = TeamAccount::LEN, seeds=[_team_name.as_bytes(), &_team_id.to_ne_bytes()], bump)]
    pub team_account: Account<'info, TeamAccount>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

We are initializing team account and assign payer as signer. init, payer and space is compulsory for initializing. For the team creation we are generating
seed for PDA Account which will be unique because team_id will be unique. 

And for the ``` space = TeamAccount::LEN ```:

We implement constant team account struct:

```rust
impl TeamAccount {
    const LEN: usize = 8 // discriminator 
    + 32 // captain pubkey 
    + 1 // bump 
    + 32 // name
    + 5 * 32 // members vector 
    + 8 // id 
}
```

<b>Note that: </b> Solana Account can hold on 10MB maximum data. You can learn more from [here](https://solanacookbook.com/core-concepts/accounts.html#facts).






