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
- At least one written program with Anchor before.

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

Sets the signer as the captain of the team and add the address as a member of the team. 

First we will create Team account struct:

```rust
#[account]
pub struct TeamAccount {
    pub captain: Pubkey,
    pub bump: u8,
    pub name: String,
    pub members: Vec<Pubkey>,
    pub id: u64,
    pub distribution_voting_result: bool,
    pub can_join_tournament: bool,
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
    + 1 // distribution_voting_result
    + 1; // can_join_tournament
}
```

Lastly, add error codes:
For the sake of simplicity, we add all Error codes! :)

```rust
#[error_code]
pub enum ErrorCode {
    #[msg("A team can contain maximum 5 members")]
    TeamCapacityFullError,
    #[msg("Invalid bump seeds")]
    InvalidBumpSeeds,
    #[msg("A team must contain at least 2 members to be able to remove a member")]
    TeamCapacityLowError,
    #[msg("Only captain can call this function")]
    NotCaptainError,
    #[msg("Member is not in the team")]
    MemberNotInTeamError,
    #[msg("Member is already in the team")]
    MemberAlreadyInTeamError,
    #[msg("Captain cannot leave the team unless he transfers the captain role to another member")]
    CaptainCannotLeaveTeamError,
    #[msg("Member is already voted for the tournament")]
    AlreadyVotedError,
    #[msg("The team has an active tournament and cannot vote for another tournament, leave the current one first")]
    AlreadyActiveTournamentError,
    #[msg("The team has no active tournament")]
    NoActiveTournamentError,
    #[msg("A team must contain 5 players to join a tournament")]
    NotEnoughPlayersError,
    #[msg("The sum of percentages must be equal to 100")]
    InvalidPercentageError,
    #[msg("Invalid member for that reward")]
    InvalidRewardError,
}
```


<b>Note that: </b> Solana Account can hold on 10MB maximum data. You can learn more from [here](https://solanacookbook.com/core-concepts/accounts.html#facts).

## Write Add member to team functionality

- A team can only have 5 players max.
- Cant add dublicate pubkey
- Only the captain of the team can add a member

We implement Add Member function below the create_team function:

```rust
pub fn add_member(
    ctx: Context<AddMember>,
    _team_name: String,
    _team_id: u64,
    member: Pubkey,
) -> Result<()> {
    let team = &mut ctx.accounts.team_account;

    // checking if the team already has 5 players if so, return error
    require!(team.members.len() < 5, ErrorCode::TeamCapacityFullError);
    // checking if the member is already in the team, if so, return error
    require!(
        !team.members.contains(&member),
        ErrorCode::MemberAlreadyInTeamError
    );
    // checkin if the signer is the captain
    require!(
        team.captain == *ctx.accounts.signer.key,
        ErrorCode::NotCaptainError
    );

    // adding member to the team
    team.members.push(member);

    Ok(())
}
```
In this function, we pass as Pubkey the member who will be added to the team.

Then we add its instruction to outside of the program macro(pub mod team_dao):

```rust
#[derive(Accounts)]
#[instruction(team_name: String, team_id: u64)]
pub struct AddMember<'info> {
    #[account(mut, seeds=[team_name.as_bytes(), &team_id.to_ne_bytes()], bump = team_account.bump)]
    pub team_account: Account<'info, TeamAccount>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

In here, in order to understand PDA please read [this essay](https://solanacookbook.com/core-concepts/pdas.html) carefully. And we find a PDA by passing seeds and bump. Seed consist of team_name and team_id. “Seeds” are optional inputs used in the add member function to derive a PDA. Bump provides an additional seed called a "bump seed" to ensure that the result is not on the Ed25519 curve. Additionally you can find more about PDA and bump from [here](https://www.brianfriel.xyz/understanding-program-derived-addresses/).

## Create Remove Member Functionality


- There must be more than 1 member in the team to remove a member
- Only the captain of the team can remove a member
- There must be a member in the team with the given pubkey parameter.

We implement remove_member function after add_member:

```rust
pub fn remove_member(
    ctx: Context<RemoveMember>,
    _team_name: String,
    _team_id: u64,
    member: Pubkey,
) -> Result<()> {
    let team = &mut ctx.accounts.team_account;

    // checking if the team has at least 2 players if not, return error
    require!(team.members.len() > 1, ErrorCode::TeamCapacityLowError);
    // checkinf if the caller is the captain of the team
    require!(team.captain != member, ErrorCode::NotCaptainError);
    // checking it the member is in the team
    require!(
        team.members.contains(&member),
        ErrorCode::MemberNotInTeamError
    );

    // checking if the member is in the team
    require!(
        team.members.contains(&member),
        ErrorCode::MemberNotInTeamError
    );

    // removing member from team
    team.members.retain(|&x| x != member);

    Ok(())
}
```
It is almost same as add member function.
Then add instructions for the function:

```rust
#[derive(Accounts)]
#[instruction(team_name: String, team_id: u64)]
pub struct RemoveMember<'info> {
    #[account(mut, seeds=[team_name.as_bytes(), &team_id.to_ne_bytes()], bump = team_account.bump)]
    pub team_account: Account<'info, TeamAccount>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

It is also almost same as add member instructions.

## Create Transfer Team Captain Role functionality

- Only the captain of the team can transfer captainship
- There must be a member with the given pubkey parameter in the team

As you can guess we will add transfer_captain function to inside pub mod team_dao program:

```rust
pub fn transfer_captain(
    ctx: Context<TransferCaptain>,
    _team_name: String,
    _team_id: u64,
    member: Pubkey,
) -> Result<()> {
    let team = &mut ctx.accounts.team_account;

    // checking if the signer is captain
    require!(
        team.captain == *ctx.accounts.signer.key,
        ErrorCode::NotCaptainError
    );
    // checking if the member is in the team
    require!(
        team.members.contains(&member),
        ErrorCode::MemberNotInTeamError
    );

    // transferring captain role
    team.captain = member;

    Ok(())
}
```

We take member parameter in order to transfer the captainship to a certain member of the team. Randomly assigning the captainship'd be a practice.

Then we add instructions of the function:

```rust
#[derive(Accounts)]
#[instruction(team_name: String, team_id: u64)]
pub struct TransferCaptain<'info> {
    #[account(mut, seeds=[team_name.as_bytes(), &team_id.to_ne_bytes()], bump = team_account.bump)]
    pub team_account: Account<'info, TeamAccount>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

Most of the instructions will be similar to these :) but for those who did not understand completely these we will explain again:

```
#[derive(Accounts)]
```
Implements an Accounts deserializer on the given struct, applying any constraints specified via inert #[account(..)] attributes upon deserialization.
It basically specifies the struct below will be account and handle deserialization of instructions and constraints.

```
#[instruction(team_name: String, team_id: u64)]
```
During execution, a program will receive a list of account data as one of its arguments, in the same order as specified during Instruction construction.
It allows the instructions in that account to be passed from the function to here as parameters.

```
#[account(mut)]
pub signer: Signer<'info>,
```

In here, Signer<'info> tells program that this parameter will be the signer of the transaction. Remember that Solana can accept more than 1 signer to its programs. Basically when we call ctx.accounts.signer in the relevant function it will be this parameter.

## Create Leave Team Functionality


# References
https://docs.rs/solana-program/latest/solana_program/instruction/struct.Instruction.html
https://docs.rs/anchor-derive-accounts/0.4.0/anchor_derive_accounts/derive.Accounts.html


