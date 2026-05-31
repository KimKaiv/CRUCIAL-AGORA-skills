# CRUCIAL-AGORA-skills

This repository contains skills that can be used from within e.g. Claude Code to query AGORA for listing the current array of live markets (`agora-markets`), for pulling the current prices of a specified market (`agora-market`), and for identifying potential arbitrage opportunities between 1-dimensional and 2-dimensional markets (`agora-arb-opp`). 

The user will first need to develop an MCP wrapper specific to their system setup for the AGORA API.

Once MCP is working, run `agora-refresh-specs` skill to create /knowledge/specs/ folder, _index.json file, and market specification files for all currently live OPER markets. 
